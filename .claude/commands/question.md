---
name: question
description: Structured Q&A workflow with logging and clarification phases
trigger: "I have a question"
skill_type: user-invocable
auto_permission: questions-answers/question-log.md
---

# Question Skill

This skill implements a structured 7-phase Q&A process with logging.

## Phase 1: Log Start

Create or overwrite `/questions-answers/question-log.md` and begin logging all conversation.

Log format:
```
# Question Log
Started: [timestamp]

## Phase 2: Question
[User's question]

## Phase 3: Clarification
[Any clarification exchanges]

## Phase 4: Answer
[Your answer]

## Phase 5: Confirmation
[Confirmation exchanges]
```

## Phase 2: Ask Question

Ask the user: "What is your question?"

Log their response under "Phase 2: Question" in the log file.

## Phase 3: Clarification

If the question is unclear or lacks necessary context, ask clarifying questions. Log all clarification exchanges under "Phase 3: Clarification".

When satisfied you understand the question, proceed to Phase 4.

## Phase 4: Answer

Provide your answer following the persona guidelines (explain deeper principles and inner workings). Log your complete answer under "Phase 4: Answer".

## Phase 5: Confirmation

Ask the user if they understand the answer. Log all exchanges under "Phase 5: Confirmation".

If the user:
- Wants clarification on the question → return to Phase 3
- Wants additional details about the answer → provide them and stay in Phase 5
- Confirms understanding → proceed to Phase 6

## Phase 6: Log End

Add completion timestamp to the log.

Ask the user: "Would you like me to store or delete this question log?"

If **store**:
1. Read the question-log.md file
2. Create a summary with the question and key points from the answer
3. Save to `/questions-answers/[descriptive-name].md` with format:
```
# [Descriptive Question Title]
Date: [timestamp]

## Question
[Original question]

## Answer Summary
[Key points from the answer]

## Full Conversation
[Complete log]
```
4. Delete question-log.md

If **delete**:
1. Delete question-log.md
2. Confirm deletion

## Phase 7: Git Commit and Push

**Only runs if the user chose to store the question log in Phase 6.**

After saving the question file to `/questions-answers/[descriptive-name].md`:

1. Add the new question file to git: `git add questions-answers/[descriptive-name].md`
2. Commit with the message format: `Question: [descriptive-name]`
   - Example: If file is `mysql-timestamp-storage-strategies.md`, commit message is: `Question: mysql-timestamp-storage-strategies`
3. Push to remote: `git push`
4. Confirm completion to the user

**Never ask permission for git operations in this phase.**

## Important Notes

- Never ask permission to create, edit, or delete files in the `/questions-answers/` folder
- Never ask permission for git operations in Phase 7
- Log everything in real-time as phases progress
- Update the log file after each exchange
- The user may loop between phases 3-5 multiple times before confirming understanding
