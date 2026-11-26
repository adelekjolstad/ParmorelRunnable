# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

PARMOREL (Personalized and Automatic Repair of MOdels using REinforcement Learning) repairs broken EMF/Ecore models with Q-learning. Users pick preferences (e.g. punish deletion, prefer short action sequences); the algorithm explores repair sequences and returns the best-rewarded one. Results are non-deterministic (random exploration).

This repo is a **master-thesis fork** of `MagMar94/ParmorelRunnable`. Rules for working here:
- All branches and PRs go to the fork (`adelekjolstad/ParmorelRunnable`, and `adelekjolstad/projectparmorel` for core code) — never to MagMar94. `gh repo set-default` is configured for both; always pass `--repo adelekjolstad/...` to `gh pr create` anyway.
- One branch per change (`fix/...`, `setup/...`), commit messages explain what was broken, why, and how it was fixed (they serve as the thesis changelog). Keep changes to upstream code minimal and traceable.
- Tag `upstream-baseline` marks the untouched upstream state.

## Repository layout (two-level submodules)

- `src/hvl/projectparmorel/` — the runnable wrapper: `main/GUI.java` (Swing app), `main/Main.java` (batch evaluation over `mutants/`), `main/TestDifferentModelSelectionMethods.java` + `main/strategies/` (thesis experiments comparing solution-selection strategies), `scripts/` (log readers / evaluation helpers).
- `projectparmorel/` — **git submodule** (fork of `MagMar94/projectparmorel`) with the core algorithm. Changes to it are committed in the submodule repo, then the submodule pointer is bumped here.
- `projectparmorel/SimilarityMetamodelsChecker/` — nested submodule (still `MagMar94/SimilarityMetamodelsChecker`) used to compute model distance via Epsilon ECL.
- Data folders: `mutants/` (100 AMOR-mutated Ecore models for `Main`), `broken models/`, `differentSelectionMethodsOriginals/`, `model/quality.model` + `eol/` (quality metrics evaluated with Epsilon), `fixed/` (output, gitignored).

After cloning: `git submodule update --init --recursive`.

## Core architecture (projectparmorel/src)

- `ModelFixer` / `Solution` — public API. `QModelFixer` is the abstract Q-learning loop; `ecore/EcoreQModelFixer` is the only concrete implementation (it supplies `EcoreErrorExtractor`, `EcoreActionExtractor`, `EcoreModelProcessor`).
- `fixModel(File)`: copies the model, extracts errors (EMF `Diagnostician` codes), runs 25 episodes; each episode works on its own temp copy (`<parent>parmorel_temp_solution_<n>_<name>` — note: no path separator, so temp files land *next to* the model's folder). Each episode picks actions via the Q-table (epsilon-greedy, `randomFactor` 0.25) until no errors remain. Valid sequences become `QSolution`s; the best-weighted one is returned (`Solution.getModel()` is the repaired file — the input file itself is not modified).
- `reward/` — each `Preference` adds reward per action or per finished solution (`PostRepairPreference`, `SolutionPreference`); `PreferenceOption` maps GUI/strategy IDs to preferences. Quality preferences (maintainability, understandability, complexity, reuse, relaxation) run `it/gssi/cs/quality/QualityEvalEngine` (Epsilon EOL on `model/quality.model`); distance preference uses `EcoreSolution` → `EComparator` (ECL).
- `knowledge/` — Q-table and preference weights, persisted to `knowledge.xml` in the working directory (gitignored). Delete it to reset learned knowledge.
- Error codes 4 and 6 are registered as unsupported by callers (`ModelType.ECORE.addUnsupportedErrorCode`). Errors without an action (e.g. 42, 51) are added to unsupported at runtime, so some models (e.g. `broken models/ATL.ecore`) yield no solution.

## Building and running

Developed in IntelliJ on macOS with JDK 21 (manifest says JavaSE-13; upstream targeted Eclipse PDE). The `.iml` and `.idea/` are local and gitignored. Source roots: `src`, `projectparmorel/src`, `projectparmorel/test`, `projectparmorel/SimilarityMetamodelsChecker/{epsilon,src,support}`. Libraries: all jars in `projectparmorel/libs/` and `projectparmorel/SimilarityMetamodelsChecker/required/`, plus two that used to come from Eclipse:
- `junit-platform-commons` (production code uses `ExceptionUtils.readStackTrace`) — from `~/.m2/repository/org/junit/platform/junit-platform-commons/1.9.1/`
- `org.eclipse.epsilon.eol.dt` (used by `EpsilonStandaloneExample`) — from `/Applications/Eclipse.app/Contents/Eclipse/plugins/org.eclipse.epsilon.eol.dt_2.9.0.*.jar`

Command-line compile (no build tool exists):
```bash
CP=$(ls projectparmorel/libs/*.jar projectparmorel/SimilarityMetamodelsChecker/required/*.jar | tr '\n' ':')$HOME/.m2/repository/org/junit/platform/junit-platform-commons/1.9.1/junit-platform-commons-1.9.1.jar:$(ls /Applications/Eclipse.app/Contents/Eclipse/plugins/org.eclipse.epsilon.eol.dt_2.9.0.*.jar)
find src projectparmorel/src projectparmorel/SimilarityMetamodelsChecker/{epsilon,src,support} -name '*.java' > /tmp/sources.txt
javac -nowarn -encoding UTF-8 -d out -cp "$CP" @/tmp/sources.txt
```

Run from the **repo root** (paths like `././mutants/`, `model/quality.model` and `knowledge.xml` are relative to the working directory). The source folders must also be on the runtime classpath, because `.ecl`/`.eol` scripts are loaded as classpath resources:
```bash
RCP="out:src:projectparmorel/src:projectparmorel/SimilarityMetamodelsChecker/epsilon:projectparmorel/SimilarityMetamodelsChecker/src:projectparmorel/SimilarityMetamodelsChecker/support:$CP"
java -cp "$RCP" hvl.projectparmorel.main.GUI      # interactive: import → choose preferences → repair → export
java -cp "$RCP" hvl.projectparmorel.main.Main     # repairs every model in mutants/ into fixed/
```
`Main` fails on models already present in `fixed/` — clear it before re-running. The GUI writes its working copy as `repaired-<name>` next to the imported model and fails if that file already exists.

Tests (`projectparmorel/test`, JUnit 5 Jupiter; only knowledge/Q-table classes are covered) are run from IntelliJ; they need `junit-jupiter-api` on the classpath.
