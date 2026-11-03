# Code Merge Agent

A VS Code agent (GitHub Copilot custom agent) that helps resolve Git merge conflicts, one file at a time. Built as an internship project. Status: working prototype, see "Known limits".

## What it does

You give it the name of one conflicted file. It reads the conflict and returns one of three outcomes:

| Outcome | Meaning |
|---|---|
| AUTO-APPLY | Edits the file. Only under Rule 1a: one side is a code-level superset of the other (the other side lacks only comments or formatting), checked by a diff and a file hash. |
| SUGGEST | Proposes a resolution with the combined code and an explanation. The developer applies it. |
| ABSTAIN | Declines and explains why, for example when the conflict is ambiguous or a cherry-picked commit depends on code missing from the current branch. |

Abstaining is a valid result, not a failure.

It handles both merges and cherry-picks. It never stages or commits: the file is left unstaged for the developer to review.

## How to use it

### Requirements
- VS Code with GitHub Copilot Chat
- The GitHub MCP server connected in VS Code. This is required: the agent uses it to check the remote copy of the file.
- The repo's `origin` points to GitHub, and the branches involved in the merge or cherry-pick are pushed. Unpushed local commits can make the remote check fail, and the agent will then abstain and say why.

### Setup
1. Copy `.github/agents/codemergeagent.agent.md` into the `.github/agents/` folder of the repo you work in.
2. In VS Code, run **Reload Window** from the Command Palette (Ctrl+Shift+P). VS Code can keep using an old copy of the agent file until you do.
3. Open the repo folder that has the conflict.

### Run
1. Start a merge or cherry-pick that produces a conflict.
2. Open Copilot Chat and choose `codemergeagent` from the Agent dropdown.
3. Type the name of one conflicted file, for example `MyFile.java`.
4. Read the outcome (AUTO-APPLY, SUGGEST or ABSTAIN) and the explanation.
5. Approve read-only git commands (for example `git status`, `git log`, `git diff`). Refuse anything that changes git state (`git add`, `git commit`, `git merge`, `git cherry-pick`, `git reset`). The agent never needs to stage or commit.
6. When the agent is done, review the file (for example with `git diff`), compile it if it is code, then stage it yourself with `git add <file>` and continue the merge or cherry-pick. The agent leaves the file unstaged on purpose.

### After a SUGGEST
Paste the proposed code into a scratch copy of the file and compile it before applying it. In one test, a suggestion with edge errors failed to compile, while a cleaned-up version compiled with 0 errors. This is a recommended developer step, not something the agent does.

## How it decides

The agent follows four steps for every conflicted file.

1. **Gather facts.**
   - Works out whether this is a merge or a cherry-pick. A rebase or revert is not supported, so it abstains.
   - Confirms the file is really unmerged. If one side deleted the file, it abstains.
   - Reads the three versions Git keeps: base (`:1:`), ours (`:2:`) and theirs (`:3:`).
   - Uses the GitHub MCP server to check that the remote copies match the local ours and theirs. If they do not match, it abstains and says which side failed.
   - Finds the exact lines of the conflict markers.
2. **Classify the conflict.**
   - **Rule 1a (superset check), first.** Runs a whitespace-insensitive diff between ours and theirs. If every line one side lacks is only a comment or formatting, the other side is a code-level superset and wins, regardless of commit dates. The result is verified with a file hash. Skipped for cherry-picks.
   - **Base comparison, for every region.** If one side is unchanged from base, only the other side changed, so the changed side is taken.
3. **Decide one outcome** for the whole file: AUTO-APPLY, SUGGEST or ABSTAIN.
4. **Check the report** against the Git data before writing it, then answer in a fixed format (outcome, reason, proposed code, skipped commands, adjacent issues).

### Known patterns
The agent file lists conflict shapes that went wrong in testing:
1. **Hoisted shared lines.** Git moves lines identical on both sides out of the conflict. Each side then looks incomplete, for example a closing `}`.
2. **Identical lines inside a region.** Keeping both sides must keep that line once, not twice.
3. **One-sided change inside a bigger region.** A line sits in the region only because it is next to a real conflict. Take the changed side.
4. **A value neither side has.** For example a new version number. The agent abstains and states the human decision.
5. **Missing dependency (cherry-pick).** The picked commit uses code that exists only in an earlier commit not on the current branch. The agent abstains, lists the likely missing commits, and never copies that code in.

## Testing

Results come from different versions of the agent file (patch numbers are shown). The published file is patch 8.1. Work assigned by the manager is kept separate from our own additions. The full run log is kept outside this repo because it quotes code from the kit.

### Assigned: the manager's fixture kit
- 27 numbered runs across the 4 conflicting files of the kit. The expected outcome was written down before each run (except Run 1, which was logged afterwards from a saved copy of its result), so the runs were tests and not demonstrations.
- The agent file went through 8 patches, each aimed at a failure seen in the runs.
- Patch 6 repeat series (3 runs per file, 12 runs, same model, one Flash run excluded):

| File (manager's kit) | Decisions correct | Reports fully accurate |
|---|---|---|
| File 1 | 2/3 | 3/3 |
| File 2 | 3/3 | 3/3 |
| File 3 | 1/3 | 3/3 |
| File 4 | 2/3 | 3/3 |
| **Total** | **8/12** | **12/12** |

- Safety in that series: 0 file edits.
- Reports: 12/12 passed all six report checks. Before patch 6 the best result was 4 of 6.
- Remaining failure types: dropped or duplicated lines at the edge of a conflict block, picking a value instead of abstaining, offering "ours or theirs" where both could be combined, and an incomplete list of added lines.
- Patch 7 tried to fix the edge errors. It tested worse (3 of 3 runs partly wrong) and was rolled back.

### Our own additions (not assigned)
- **Spot-check on our own scenarios.** One run each on imports and methods, same method with different parameters, and a pom.xml version and dependency conflict, using code the agent was never tuned on (patch 6). Result: 3 of 3 correct.
- **Cherry-pick support (patch 8).** Two scenarios, tested on GPT-6 Luna:

| Scenario | Result |
|---|---|
| Diverged values | Correct (SUGGEST, ABSTAIN on the repeat) |
| Missing dependency | Correct (ABSTAIN, both runs) |

- **Compile check.** On one patch 7 suggestion with edge errors, compiling a pasted copy failed, and a cleaned-up version compiled with 0 errors. This is one test, not a full series. It is a recommended developer step and is not built into the agent.

### Not yet tested
- Merges on the final file (patch 8.1) were not re-run. The merge series ran on patch 6.
- Six of our own scenarios (scn1–5 and scn8–9) have not been run.

## Known limits

- **Safety depends on the model.** On a paid GitHub Copilot plan you can choose one model. On our plan only "Auto" was available, so the model changed between runs. MAI Code 1.1 Flash broke the hard rules twice: once it skipped a required check instead of abstaining, and once it edited a file during a cherry-pick and copied in code from a dependency commit, then reported the outcome wrongly. GPT-6 Luna did not do this in the runs we recorded, but earlier runs on the unpatched agent and an earlier model did try forbidden commands such as `git add`. Check which model answered, and re-run if it was Flash.
- **Edit approval is unconfirmed.** Terminal commands asked for approval in our setup. During testing, file edits appeared to go through without a prompt, but we have not re-checked this. Until you have checked your own setup, treat edits as ungated: the agent's own rule-following may be the only safeguard.
- **The agent's report is advisory.** Git state is the source of truth: check the file itself, not the agent's summary of what it did.
- **Decisions are not always right.** In the patch 6 series 8 of 12 decisions were fully correct. Known failure types: dropped or duplicated lines at conflict edges, picking a value instead of abstaining, and "ours or theirs" answers where both sides could be combined. Compile suggested code before applying it.
- **Needs GitHub.** The remote check requires the GitHub MCP server, a GitHub `origin`, and pushed branches.
- **Limited test coverage.** Merges on the final file (patch 8.1) were not re-run, and six of our own scenarios have not been run. Testing used Java files and one pom.xml. Scala was not tested.
- **Tested on one kit.** The patches were shaped around the manager's four files, so results on other code may differ. The spot-check on our own scenarios (3 of 3 correct) is a small sample.
