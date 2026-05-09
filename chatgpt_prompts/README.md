# ChatGPT Prompt Exports

Each file is a standalone ChatGPT web prompt adapted from one Academic Research Skills module.

## Files

1. `academic_paper_prompt.md` for `academic-paper`
1. `academic_paper_reviewer_prompt.md` for `academic-paper-reviewer`
1. `academic_pipeline_prompt.md` for `academic-pipeline`
1. `deep_research_prompt.md` for `deep-research`

## Use

1. Open ChatGPT web with GPT Pro.
2. Paste one prompt file as the first message in a new chat.
3. Wait for the model to acknowledge the role.
4. Send the actual research, writing, review, or pipeline task in the next message.

The exports are adapted for ChatGPT web. Local platform names, plugin loading, hooks, shell assumptions, local report paths, and agent routing are removed or converted into conversational workflow rules.

Technical provenance blocks, schema comments, audit templates, byte counts, and hashes are retained only to preserve source traceability. They are not part of the normal runtime behavior in ChatGPT web.

The `academic_pipeline_prompt.md` file is an orchestrator prompt. For a full stage execution in ChatGPT web, pair it with the corresponding standalone prompt for `deep-research`, `academic-paper`, or `academic-paper-reviewer` when the pipeline reaches that stage.
