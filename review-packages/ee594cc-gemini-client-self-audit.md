# Review package: niklar-stocks commit ee594cc

Self-audit finding + fix in `scripts/ai_qa/gemini_client.py`, published
here in full so ChatGPT's GitHub connector -- which returned 404 on the
private `niklar-stocks` repo directly -- can independently inspect the
exact code/diff without needing private-repo access. Requested in the
Drive sync doc's `RESPONSE_SEQ: 1` entry (2026-08-09): "Claude provides
a narrowly scoped, privacy-safe review package through an already-
authorized accessible channel that contains only the exact non-
sensitive code/diff needed for independent review."

**Scope**: exactly the 3 files this commit touched. Nothing else from
`niklar-stocks` is reproduced here. Content below was scanned for
secrets and Niklar canonical/proprietary content before publishing to
this public repo (clean -- the one `AIza...`-looking string in the test
file is a synthetic, obviously-fabricated fixture used to test that
the redaction path scrubs Google-API-key-shaped strings, not a real
key). This module (`scripts/ai_qa/`) is dev/CI tooling that runs
outside the Niklar authority chain entirely -- it executes existing
test/lint/build commands and talks to the Gemini API over stdlib
`urllib`; it contains no trading, scoring, or canonical business logic.

## Commit message (verbatim)

```
commit ee594cc500971684af3a1911be19031a1a5310c8
Author: Claude <noreply@anthropic.com>
Date:   Sat Aug 8 18:25:42 2026 +0000

    Self-audit: fix uncaught mid-response network error in gemini_client.py
    
    Per ChatGPT's newly frozen independent engineering review pattern
    (Drive sync doc, 2026-08-09 01:40 +08:00 entry), naming scripts/ai_qa/
    as the first reference case: Claude performs its own simplification/
    self-audit, then publishes a compact review handoff for ChatGPT to
    independently inspect. This commit is that self-audit's real finding
    and fix, found by reading the code -- not by a live failure.
    
    FINDING: gemini_client.diagnose()'s try/except around the Gemini HTTP
    call only handled HTTPError/URLError/TimeoutError. All three cover
    failures during connection setup. A connection that succeeds but is
    reset (or returns an incomplete body) *while streaming the response* --
    e.g. ConnectionResetError, or http.client.IncompleteRead, which is not
    even an OSError subclass -- was uncaught. Since run.py's main() only
    ever catches GeminiError, that failure mode would propagate past this
    module entirely and crash the whole worker, losing the deterministic
    FAIL report the run had already earned before Gemini was even invoked.
    This directly contradicted the module's own documented guarantee
    ("never an unhandled traceback").
    
    FIX: added `except (OSError, http.client.HTTPException)` after the
    existing three, mapped to GeminiError like every other Gemini-side
    failure mode -- same graceful-degrade path, no new behavior class.
    
    TEST COVERAGE: gemini_client.py had zero automated tests before this
    commit -- only ad-hoc manual verification against the real endpoint
    (documented in README.md). That gap is exactly how a bug like this
    stays invisible. Added tests/test_ai_qa_gemini_client.py: 14 tests
    covering the happy path, every network-failure mode (including two
    regression tests for this exact finding -- ConnectionResetError and
    IncompleteRead, which needed separate cases since only one is an
    OSError subclass), and every malformed-response shape the code already
    handles (blockReason, MAX_TOKENS, zero candidates, non-JSON body,
    non-JSON structured text, missing required field). All mocked -- no
    real network call in the test suite.
    
    Reviewed the rest of scripts/ai_qa/ against the same criteria (Minimum
    Sufficient Code, unnecessary abstraction/dependency, duplicate logic,
    security/privacy leakage, architecture/canon drift) -- no other
    findings. redact.py/failure_extract.py/run.py/config.py are unchanged.
    
    Full validation: unittest 95/95 (81->95, 14 new), mypy 0/13, PWA smoke
    unaffected, worker run clean end-to-end locally.
    
    Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
    Claude-Session: https://claude.ai/code/session_01Fd6zhPr8XrRzdMckvrXZQJ
```

## Diff: `scripts/ai_qa/gemini_client.py` and `scripts/ai_qa/README.md`

```diff
diff --git a/scripts/ai_qa/README.md b/scripts/ai_qa/README.md
index b8b91f7..1100c98 100644
--- a/scripts/ai_qa/README.md
+++ b/scripts/ai_qa/README.md
@@ -209,6 +209,24 @@ Beyond the two points above:
   which resolves normally) — caught explicitly in `gemini_client.py`,
   and the error body is run through `sanitize_output()` before it can
   reach the report, same discipline as everywhere else.
+- **A connection reset/incomplete read *during* the response body is a
+  distinct failure mode from `URLError`** (which only covers failures
+  during connection setup, before `urlopen()` returns a response
+  object) — found by a self-audit (2026-08-09) that read this module
+  looking for gaps against its own "never an unhandled traceback"
+  claim, not by a live failure. Without an explicit `except (OSError,
+  http.client.HTTPException)` around the response read, that failure
+  mode would propagate straight past `gemini_client.py` and crash the
+  whole worker — `run.py`'s `main()` only ever catches `GeminiError` —
+  losing the deterministic `FAIL` report the run had already earned.
+  Fixed, with a regression test for both the `OSError`-subclass case
+  (`ConnectionResetError`) and the non-`OSError` case
+  (`http.client.IncompleteRead`) — see
+  `tests/test_ai_qa_gemini_client.py`, which also closed this module's
+  prior zero-automated-test-coverage gap in the same pass (14 new
+  tests covering the full happy/network-failure/malformed-response
+  matrix; previously only manually verified against the real
+  endpoint).
 - **`TOOL_MISSING` vs. `NOT_CONFIGURED` are structurally distinct.** A
   missing `mypy` install is a hard `FAIL` (`run.py`'s `_preflight_ok`),
   never reported the same way as the deliberately-skipped `lint`/`build`
diff --git a/scripts/ai_qa/gemini_client.py b/scripts/ai_qa/gemini_client.py
index 5751fcd..dec4dbe 100644
--- a/scripts/ai_qa/gemini_client.py
+++ b/scripts/ai_qa/gemini_client.py
@@ -26,6 +26,7 @@ oversights -- see README.md "Python-specific correctness fixes"):
 """
 from __future__ import annotations
 
+import http.client
 import json
 import urllib.error
 import urllib.request
@@ -151,6 +152,19 @@ def diagnose(step_label: str, command: str, exit_code: int, diagnostic_payload:
         raise GeminiError(f"Gemini API unreachable: {exc.reason}") from None
     except TimeoutError:
         raise GeminiError(f"Gemini API timed out after {config.GEMINI_TIMEOUT_SECONDS}s") from None
+    except (OSError, http.client.HTTPException) as exc:
+        # Covers failures mid-response-read (connection reset, incomplete
+        # body) -- distinct from URLError, which only wraps failures during
+        # connection setup, before urlopen() returns a response object.
+        # Without this, such a failure propagates past this function
+        # entirely: run.py's main() only catches GeminiError, so the
+        # worker would crash instead of degrading to the same graceful
+        # FAIL-with-no-diagnosis path as every other Gemini-side failure
+        # mode -- contradicting this module's own "never an unhandled
+        # traceback" guarantee (see module docstring). Found by code
+        # review, not by a live failure -- see
+        # tests/test_ai_qa_gemini_client.py for the regression test.
+        raise GeminiError(f"Gemini API connection failed while reading the response: {exc}") from None
 
     try:
         parsed = json.loads(raw)
```

## New file (full): `tests/test_ai_qa_gemini_client.py`

```python
"""Tests for scripts/ai_qa/gemini_client.py.

Did not exist until this repo's independent-review self-audit (see
niklar-handoff for the checkpoint) -- ChatGPT's newly frozen review
pattern asked specifically for "missing failure-envelope coverage," and
this module (the most branching error-handling code in scripts/ai_qa/)
had none: only ad-hoc manual verification against the real endpoint
(clean pass, clean HTTP-400-with-invalid-key handling, and later a real
MECHANICAL_FIX classification -- see scripts/ai_qa/README.md). That gap
is exactly how the connection-reset-mid-response bug this file also
regression-tests (see DiagnoseNetworkFailureTests below) went unnoticed:
found by reading the code during the audit, not by a live failure.

Every network call is mocked -- no real HTTP request is made by this
file. diagnose() is exercised directly, independent of
config.gemini_enabled() (that gate lives in run.py, not here).
"""
from __future__ import annotations

import http.client
import json
import unittest
import urllib.error
from unittest.mock import MagicMock, patch

from scripts.ai_qa.gemini_client import Diagnosis, GeminiError, diagnose

_VALID_DIAGNOSIS = {
    "status": "MECHANICAL_FIX",
    "summary": "mypy type mismatch",
    "likely_cause": "wrong annotation",
    "affected_files": ["scripts/ai_qa/config.py"],
    "recommended_fix": "fix the annotation",
    "confidence": "HIGH",
}


def _mock_response(raw_bytes: bytes | None = None, read_side_effect: BaseException | None = None) -> MagicMock:
    """A mock of the object `with urllib.request.urlopen(...) as response`
    binds -- supports the context-manager protocol, and .read() either
    returns fixed bytes or raises, depending on which is under test.
    """
    response = MagicMock()
    if read_side_effect is not None:
        response.read.side_effect = read_side_effect
    else:
        response.read.return_value = raw_bytes
    response.__enter__.return_value = response
    response.__exit__.return_value = False
    return response


def _response_json(payload: dict) -> MagicMock:
    return _mock_response(json.dumps(payload).encode("utf-8"))


def _response_with_candidate_text(text: str, finish_reason: str = "STOP") -> MagicMock:
    payload = {"candidates": [{"finishReason": finish_reason, "content": {"parts": [{"text": text}]}}]}
    return _response_json(payload)


class DiagnoseHappyPathTests(unittest.TestCase):
    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_valid_response_round_trips_into_a_diagnosis(self, mock_urlopen):
        mock_urlopen.return_value = _response_with_candidate_text(json.dumps(_VALID_DIAGNOSIS))
        result = diagnose("typecheck", "python3 -m mypy", 1, "some output")
        self.assertIsInstance(result, Diagnosis)
        self.assertEqual(result.status, "MECHANICAL_FIX")
        self.assertEqual(result.confidence, "HIGH")
        self.assertEqual(result.affected_files, ["scripts/ai_qa/config.py"])

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_missing_optional_affected_files_defaults_to_empty_list(self, mock_urlopen):
        body = {k: v for k, v in _VALID_DIAGNOSIS.items() if k != "affected_files"}
        mock_urlopen.return_value = _response_with_candidate_text(json.dumps(body))
        result = diagnose("smoke", "python3 pwa/smoke_test.py", 1, "some output")
        self.assertEqual(result.affected_files, [])


class DiagnoseNetworkFailureTests(unittest.TestCase):
    """Every network-layer failure mode must degrade to GeminiError --
    never an unhandled traceback -- matching this module's own stated
    guarantee (see its docstring).
    """

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_http_error_becomes_gemini_error_with_sanitized_body(self, mock_urlopen):
        error = urllib.error.HTTPError(
            url="https://example.invalid",
            code=400,
            msg="Bad Request",
            hdrs=None,  # type: ignore[arg-type]
            fp=MagicMock(read=lambda: b'{"error": "invalid key AIzaSyABCDEFGHIJKLMNOPQRSTUVWXYZ012345"}'),
        )
        mock_urlopen.side_effect = error
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("HTTP 400", str(ctx.exception))
        self.assertNotIn("AIzaSyABCDEFGHIJKLMNOPQRSTUVWXYZ012345", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_url_error_becomes_gemini_error(self, mock_urlopen):
        mock_urlopen.side_effect = urllib.error.URLError("name resolution failed")
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("unreachable", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_timeout_becomes_gemini_error(self, mock_urlopen):
        mock_urlopen.side_effect = TimeoutError("timed out")
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("timed out", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_connection_reset_mid_response_becomes_gemini_error(self, mock_urlopen):
        """Regression test for the self-audit finding: a connection that
        succeeds but is reset while streaming the body (as opposed to
        failing during connection setup, which URLError already covers)
        used to propagate straight past this module and crash the
        worker -- run.py's main() only ever catches GeminiError, so the
        deterministic FAIL report would never get written.
        """
        mock_urlopen.return_value = _mock_response(read_side_effect=ConnectionResetError("connection reset by peer"))
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("connection failed", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_incomplete_read_becomes_gemini_error(self, mock_urlopen):
        """http.client.IncompleteRead is NOT an OSError subclass -- a
        distinct case from the ConnectionResetError regression above,
        both closed by the same except clause.
        """
        mock_urlopen.return_value = _mock_response(read_side_effect=http.client.IncompleteRead(b"partial"))
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("connection failed", str(ctx.exception))


class DiagnoseMalformedResponseTests(unittest.TestCase):
    """The Gemini API can itself misbehave in ways that aren't network
    errors -- these must all degrade to GeminiError too, never a crash.
    """

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_non_json_top_level_response(self, mock_urlopen):
        mock_urlopen.return_value = _mock_response(b"not json at all")
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("non-JSON", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_block_reason_becomes_gemini_error(self, mock_urlopen):
        payload = {"promptFeedback": {"blockReason": "SAFETY"}, "candidates": []}
        mock_urlopen.return_value = _response_json(payload)
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("blocked", str(ctx.exception))
        self.assertIn("SAFETY", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_zero_candidates_becomes_gemini_error(self, mock_urlopen):
        mock_urlopen.return_value = _response_json({"candidates": []})
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("zero candidates", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_max_tokens_finish_reason_becomes_gemini_error(self, mock_urlopen):
        mock_urlopen.return_value = _response_with_candidate_text('{"status": "tr', finish_reason="MAX_TOKENS")
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("MAX_TOKENS", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_missing_text_part_becomes_gemini_error(self, mock_urlopen):
        payload = {"candidates": [{"finishReason": "STOP", "content": {"parts": []}}]}
        mock_urlopen.return_value = _response_json(payload)
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("no text content", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_structured_text_not_valid_json_becomes_gemini_error(self, mock_urlopen):
        mock_urlopen.return_value = _response_with_candidate_text("this is not json")
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("not valid JSON", str(ctx.exception))

    @patch("scripts.ai_qa.gemini_client.urllib.request.urlopen")
    def test_missing_required_field_becomes_gemini_error(self, mock_urlopen):
        incomplete = {k: v for k, v in _VALID_DIAGNOSIS.items() if k != "confidence"}
        mock_urlopen.return_value = _response_with_candidate_text(json.dumps(incomplete))
        with self.assertRaises(GeminiError) as ctx:
            diagnose("typecheck", "python3 -m mypy", 1, "output")
        self.assertIn("missing a required field", str(ctx.exception))


if __name__ == "__main__":
    unittest.main()
```

## Full file for context (post-fix): `scripts/ai_qa/gemini_client.py`

Reproduced in full (not just the diff) since the finding concerns
exception-handling flow across the whole function -- useful to see the
new `except` clause in place alongside the three it sits next to.

```python
"""Thin REST client for Gemini failure diagnosis.

Ported from gemini-client.js in joiedevivre02/journey-passport (same
commit noted in config.py) -- a single POST doesn't justify a new SDK
dependency there, and the same is true here: this uses `urllib.request`
(stdlib), matching how pwa/smoke_test.py already talks HTTP elsewhere in
this repo for the same "stay stdlib-only" reason.

Diagnosis-only, by construction: the response schema has no field that
could cause a source edit, and the caller (run.py) never treats a
response from here as anything but text for the report. This module
never decides PASS/FAIL -- that stays purely a function of the
deterministic steps' exit codes, in run.py.

Differences from a literal port of the Node client (deliberate, not
oversights -- see README.md "Python-specific correctness fixes"):
 - `urllib.request` raises `urllib.error.HTTPError` on 4xx/5xx instead of
   resolving normally the way `fetch` does; caught explicitly here.
 - The response body of any error (including the redacted body of an
   HTTPError) is passed through `redact.sanitize_output` before it can
   ever reach the caller -- an unsanitized third-party string is exactly
   the kind of hole this repo's discipline doesn't allow.
 - `finishReason: MAX_TOKENS` (truncated JSON) and `promptFeedback.
   blockReason` (zero candidates) are both handled as diagnosis failures,
   not exceptions -- never an unhandled traceback.
"""
from __future__ import annotations

import http.client
import json
import urllib.error
import urllib.request
from dataclasses import dataclass

from . import config
from .redact import sanitize_output

# The off-limits list is Niklar's own -- NOT a find-replace of the source
# pattern's product-specific list. If a failure touches any of these,
# Gemini must return CLAUDE_REQUIRED rather than attempt a diagnosis.
_OFF_LIMITS = (
    "canonical Niklar trading logic",
    "scoring rules",
    "price-action rules",
    "entry logic",
    "invalidation logic",
    "support/resistance logic",
    "Decision Object semantics",
    "research hierarchy",
    "market-state classification",
    "database semantics",
    "security/privacy boundaries",
    "private deployment rules",
)

_SYSTEM_INSTRUCTION = (
    "You are a first-pass CI failure classifier for a private, non-public "
    "trading-research repository. You may diagnose mechanical problems "
    "only: test failures, lint failures, type errors, build failures, "
    "component-rendering failures, mobile/PWA regression failures, "
    "stale/partial/unknown-state rendering issues, and other mechanical "
    "UI/test problems. You must NOT propose, imply, or reinterpret any "
    "change to: " + "; ".join(_OFF_LIMITS) + ". You never edit files and "
    "your output never determines pass/fail -- you only classify and "
    "suggest, for a human or a separate coding agent to act on. If the "
    "failure touches any off-limits area above, OR you are not confident "
    "it is purely mechanical, you MUST return status CLAUDE_REQUIRED "
    "rather than attempt a diagnosis. If you cannot classify the failure "
    "at all, return UNKNOWN. Because this repository's own test suite "
    "specifically validates trading-logic renderers, schema locks, and "
    "publication rules, treat any failure inside ops/ or a test file that "
    "exercises ops/ as CLAUDE_REQUIRED by default unless the failure is "
    "obviously a trivial, non-semantic issue (e.g. an import error, a "
    "syntax error, a missing test fixture)."
)

_RESPONSE_SCHEMA = {
    "type": "OBJECT",
    "properties": {
        "status": {"type": "STRING", "enum": ["MECHANICAL_FIX", "CLAUDE_REQUIRED", "UNKNOWN"]},
        "summary": {"type": "STRING"},
        "likely_cause": {"type": "STRING"},
        "affected_files": {"type": "ARRAY", "items": {"type": "STRING"}},
        "recommended_fix": {"type": "STRING"},
        "confidence": {"type": "STRING", "enum": ["HIGH", "MEDIUM", "LOW"]},
    },
    "required": ["status", "summary", "likely_cause", "affected_files", "recommended_fix", "confidence"],
}


@dataclass(frozen=True)
class Diagnosis:
    status: str
    summary: str
    likely_cause: str
    affected_files: list[str]
    recommended_fix: str
    confidence: str


class GeminiError(Exception):
    """Raised for any Gemini-side problem. Message is always already
    secret-free -- callers may include str(exc) directly in a report.
    """


def diagnose(step_label: str, command: str, exit_code: int, diagnostic_payload: str) -> Diagnosis:
    """Sends only the minimum necessary diagnostic context: the failing
    step's label/command/exit code plus `diagnostic_payload` (already
    prepared by the caller -- either failure_extract's safe summary for
    the `test` step, or sanitize_output()'d raw output for other steps).
    Never the full repository, never the full Niklar canon, never
    secrets/credentials/database dumps -- see README.md.
    """
    prompt = (
        f"Step: {step_label}\n"
        f"Command: {command}\n"
        f"Exit code: {exit_code}\n"
        f"Diagnostic context:\n{diagnostic_payload}"
    )

    body = json.dumps(
        {
            "systemInstruction": {"parts": [{"text": _SYSTEM_INSTRUCTION}]},
            "contents": [{"parts": [{"text": prompt}]}],
            "generationConfig": {
                "temperature": 0,
                "responseMimeType": "application/json",
                "responseSchema": _RESPONSE_SCHEMA,
            },
        }
    ).encode("utf-8")

    url = f"{config.GEMINI_API_BASE}/models/{config.GEMINI_MODEL}:generateContent"
    request = urllib.request.Request(
        url,
        data=body,
        headers={
            "Content-Type": "application/json",
            "x-goog-api-key": config.GEMINI_API_KEY,
        },
        method="POST",
    )

    try:
        with urllib.request.urlopen(request, timeout=config.GEMINI_TIMEOUT_SECONDS) as response:
            raw = response.read().decode("utf-8", errors="replace")
    except urllib.error.HTTPError as exc:
        error_body = sanitize_output(exc.read().decode("utf-8", errors="replace"), config.MAX_PAYLOAD_CHARS)
        raise GeminiError(f"Gemini API returned HTTP {exc.code}: {error_body}") from None
    except urllib.error.URLError as exc:
        raise GeminiError(f"Gemini API unreachable: {exc.reason}") from None
    except TimeoutError:
        raise GeminiError(f"Gemini API timed out after {config.GEMINI_TIMEOUT_SECONDS}s") from None
    except (OSError, http.client.HTTPException) as exc:
        # Covers failures mid-response-read (connection reset, incomplete
        # body) -- distinct from URLError, which only wraps failures during
        # connection setup, before urlopen() returns a response object.
        # Without this, such a failure propagates past this function
        # entirely: run.py's main() only catches GeminiError, so the
        # worker would crash instead of degrading to the same graceful
        # FAIL-with-no-diagnosis path as every other Gemini-side failure
        # mode -- contradicting this module's own "never an unhandled
        # traceback" guarantee (see module docstring). Found by code
        # review, not by a live failure -- see
        # tests/test_ai_qa_gemini_client.py for the regression test.
        raise GeminiError(f"Gemini API connection failed while reading the response: {exc}") from None

    try:
        parsed = json.loads(raw)
    except json.JSONDecodeError:
        raise GeminiError("Gemini API returned a non-JSON response") from None

    prompt_feedback = parsed.get("promptFeedback") or {}
    block_reason = prompt_feedback.get("blockReason")
    if block_reason:
        raise GeminiError(f"Gemini blocked the request: {block_reason}")

    candidates = parsed.get("candidates") or []
    if not candidates:
        raise GeminiError("Gemini returned zero candidates")

    candidate = candidates[0]
    finish_reason = candidate.get("finishReason")
    if finish_reason == "MAX_TOKENS":
        raise GeminiError("Gemini response was truncated (MAX_TOKENS) before completing structured output")

    parts = (candidate.get("content") or {}).get("parts") or []
    if not parts or "text" not in parts[0]:
        raise GeminiError(f"Gemini returned no text content (finishReason={finish_reason})")

    try:
        diagnosis_json = json.loads(parts[0]["text"])
    except json.JSONDecodeError:
        raise GeminiError("Gemini's structured output was not valid JSON") from None

    try:
        return Diagnosis(
            status=diagnosis_json["status"],
            summary=diagnosis_json["summary"],
            likely_cause=diagnosis_json["likely_cause"],
            affected_files=list(diagnosis_json.get("affected_files") or []),
            recommended_fix=diagnosis_json["recommended_fix"],
            confidence=diagnosis_json["confidence"],
        )
    except KeyError as exc:
        raise GeminiError(f"Gemini's structured output was missing a required field: {exc}") from None
```

## Independent review checklist (from the frozen pattern)

- **Minimum Sufficient Code**: one new import (`http.client`), one new
  `except` clause (6 lines of code + comment), mapped to the same
  `GeminiError` path every other failure mode already used. No new
  abstraction introduced.
- **Unnecessary abstraction/dependency**: none added -- `http.client`
  is stdlib, already transitively imported via `urllib.request`.
- **Duplicate logic**: none -- reuses the existing `GeminiError`
  raise-and-degrade pattern verbatim.
- **Missing failure-envelope coverage**: this *is* the finding --
  `ConnectionResetError` (an `OSError` subclass) and
  `http.client.IncompleteRead` (not an `OSError` subclass, hence why
  the except clause needs both types) during the response-body read
  were previously uncaught. `run.py`'s `main()` only catches
  `GeminiError` from `diagnose()`, so either exception would have
  propagated past this module and crashed the worker process --
  losing the deterministic FAIL report already produced by the
  earlier failing validation step. Regression tests:
  `test_connection_reset_mid_response_becomes_gemini_error` and
  `test_incomplete_read_becomes_gemini_error` in the test file above.
- **Security/privacy leakage**: none -- the new except clause's error
  message is `str(exc)` of an `OSError`/`HTTPException`, which never
  embeds request headers, the API key, or response content.
- **Architecture/canon drift**: none -- `scripts/ai_qa/` remains
  outside the Niklar authority chain (see its own README.md "Why
  scripts/, not ops/"); this change touches only HTTP-transport error
  handling for the Gemini diagnostic call, nothing trading/canon
  related.

## Real-world verification (not just local)

Real GitHub Actions run for this commit: `31271910267`
(`joiedevivre02/niklar-stocks`, workflow `ai-qa.yml`) --
`status: "completed"`, `conclusion: "success"`. Local: `unittest`
95/95 (81->95, the 14 new tests above), `mypy` 0 issues across 13
files, PWA smoke test unaffected.
