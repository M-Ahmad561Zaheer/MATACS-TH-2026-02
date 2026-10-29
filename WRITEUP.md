# Technical Assessment: AI Proposal Triage Design

## 1. AI vs. Rules

**AI-Based Steps**

- **Structured Data Extraction:** An LLM can read proposal PDFs and extract relevant information into a structured JSON format based on the criteria configured by the admin. This is useful when proposals contain unstructured text and the evaluation requirements can change.
- **Qualitative Scoring:** AI can evaluate criteria such as team capability, proposed approach, and relevant experience. It can assign scores based on the provided guidelines and explain the reasons behind those scores.

**Rule-Based Steps**

- **Prompt Injection Detection:** Use regex and deterministic checks to identify suspicious instructions or possible prompt injection attempts in proposal text, including hidden instructions in appendices.
- **Mandatory Criteria Evaluation:** Apply fixed business rules to check eligibility. For example, if an electrical licence is mandatory and the proposal does not meet this requirement, the system marks it as ineligible.
- **Weighted Scoring and Calculations:** All numerical calculations, including price normalization, warranty conversion, and final weighted scores, are handled in code. For example, the final score is calculated using the sum of each criterion's weight multiplied by its sub-score.

**Why this approach?**

AI is useful for understanding text and evaluating qualitative information, but it should not be responsible for enforcing strict eligibility conditions or performing critical calculations. Keeping these operations in code makes the evaluation more consistent and easier to verify.

## 2. Consistency and Explainability

**Repeatability**

- Use `temperature: 0.0` and structured JSON outputs for extraction and qualitative scoring to reduce variation in responses.
- Store the extracted data, scoring parameters, and criteria used for each evaluation in the database so that the results can be traced back to their original configuration.

**Explainability**

- Each AI-generated score should include a `reasoning` field explaining why the score was assigned, supported by specific information from the proposal.
- The frontend should display a breakdown of each evaluated criterion, including the original text excerpt, extracted value, applicable rule or AI score, and final sub-score calculation.

This allows reviewers to understand how each result was produced instead of relying on a score without supporting evidence.

## 3. Human Review and Audit Logging

**Human Review Checkpoints**

- **Before Ranking:** A reviewer must verify the extracted information, particularly important fields such as cost, project timeline, and licence status, before the system generates the final ranking.
- **Flagged Proposals:** Proposals with possible prompt injection attempts, missing mandatory information, or low-confidence extractions should be marked as `flagged` and require manual review before proceeding.
- **Final Award:** The system should provide ranked recommendations only. The final contract or award decision must remain under explicit human control.

**Audit Logging**

The system should maintain an immutable audit trail containing the admin criteria version, original PDF hash, extracted JSON data, prompts and model responses, reviewer overrides, and final approval records or signatures.

This makes it possible to investigate how a proposal was evaluated and identify any changes made during the review process.

## 4. Handling Changing Criteria

**Versioned Evaluation Pipeline**

- Store each admin criteria configuration as a separate version, such as `Criteria_v1.0`.
- Every evaluation should remain linked to the criteria version used when it was originally scored, for example, `evaluated_with_version: v1.0`.
- When an admin changes the evaluation criteria or weights, existing proposals should be marked as `Re-evaluation Required` if the changes affect their evaluation.

The reviewer can either retain the historical rankings for reference or initiate a new evaluation using the updated criteria. Previous results should remain available so that the impact of the changes can be compared.

## 5. Risks and Safeguards

**Prompt Injection and Proposal Manipulation**

Proposal content must be treated as untrusted input. The system should separate proposal text from system instructions, use clear input boundaries such as `<proposal_text>`, and apply appropriate instruction hierarchy and security checks.

Regex and other deterministic checks can help identify suspicious patterns, but they cannot detect every possible prompt injection. Therefore, flagged content should be reviewed, and the model should be instructed to treat all proposal content as data rather than instructions.

**Hallucinations and Unsupported Facts**

Every extracted field should include a `quote_source` containing the exact text supporting the extracted value. The system should verify that the quote exists in the source text. If the evidence cannot be verified, the field should be set to `null` and the proposal flagged for review.

**Extraction Errors**

Fields with low confidence, missing values, or conflicting information should be flagged for manual verification rather than being accepted automatically.

**Cost and Latency**

Use small, cost-effective models, such as GPT-4o-mini or Claude 3 Haiku, for extraction and other suitable tasks. Prompt caching can also reduce repeated processing costs when the same administrative criteria and instructions are used across multiple proposals.