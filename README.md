# Code Smells Detection and Refactorings Using Large Language Models

A Java-based GitHub Pull Request reviewer that uses Large Language
Models (LLMs) to detect code smells and recommend refactorings, with
structured output validation and a reproducible evaluation framework.

Developed as my Diploma Thesis in Computer Science & Engineering at the
University of Ioannina.

## Overview

The bot analyzes Java code changed in a GitHub Pull Request, constructs
a controlled prompt for an LLM, parses the response into structured
findings, validates those findings against the analyzed source code and
diff, and generates a review-oriented report.

The repository also includes a local evaluation framework that reuses
the same analysis pipeline on labeled benchmark datasets. This makes it
possible to evaluate code-smell detection and refactoring
recommendations using precision, recall, and F1-score.

The system is designed as a developer-assistance tool rather than an
autonomous refactoring engine: it recommends refactorings but does not
automatically rewrite source code.

## Key Features

-   GitHub Pull Request and Java diff analysis
-   LLM-assisted detection of six code-smell categories
-   Refactoring recommendations
-   Constrained prompts and machine-readable JSON findings
-   Validation and source-code anchoring of LLM output
-   File-level and diff-level analysis modes
-   Line-based and method/class target-based validation
-   Local benchmark evaluation with precision, recall, and F1-score
-   GitHub Actions integration
-   JUnit tests and JaCoCo coverage

## How It Works

    GitHub Pull Request
            |
            v
      GitHubPrClient
            |
            v
    AnalyzedFileBuilder
            |
            +----> Java source
            +----> Pull-request diff
            +----> Related context
            |
            v
      PromptRenderer
            |
            v
        LLM Client
            |
            v
     Structured JSON
            |
            v
     FindingValidator
            |
            v
      ReportRenderer
            |
            v
     Review Report

The local evaluator follows a parallel workflow:

    Benchmark Dataset
            |
            v
    EvaluationDatasetLoader
            |
            v
     Shared Analysis Core
            |
            v
    EvaluationScorer
            |
            v
    Precision / Recall / F1

## Supported Code Smells

  Code Smell            Typical Refactoring Direction
  --------------------- ---------------------------------------
  Long Method           Extract Method
  Long Parameter List   Context-dependent
  Duplicate Code        Extract Method / Form Template Method
  Large Class           Extract Class
  Feature Envy          Move Method / Move Attribute
  Message Chains        Context-dependent

The taxonomy is intentionally constrained so that prompt definitions,
validation rules, and evaluation labels remain consistent.

## Structured Output

Instead of accepting unrestricted natural-language reviews, the bot
expects structured findings from the LLM.

Example:

    {
      "file": "src/main/java/example/Example.java",
      "line": 42,
      "rule": "Long Method",
      "severity": "Major",
      "note": "The method combines multiple responsibilities.",
      "suggestedRefactoring": "Extract Method",
      "targetType": "method",
      "targetName": "processOrder"
    }

Structured output allows findings to be parsed, validated, anchored to
source code, stored as artifacts, and compared with benchmark labels.

## Analysis and Validation

The analyzer supports two main scopes:

-   diff — focuses on code changed in a pull request.
-   file — analyzes the complete primary Java file and is useful for
    whole-file or method-level benchmark datasets.

The validation layer checks LLM findings before they are reported or
evaluated. The implementation supports strategies including:

-   STRICT_LINE — anchors findings using source lines and changed-line
    information.
-   TARGET_NAME — anchors findings using the reported method or class.
-   DIFF_TARGET_NAME — combines target-based anchoring with diff
    constraints.

This distinction is important because a model may identify the correct
problematic method or class while returning an imprecise line number.

## Evaluation Framework

The repository includes a local evaluator that runs labeled Java cases
through the same core analysis pipeline used by the GitHub workflow.

Evaluation material used during the thesis includes:

-   SmellyCode
-   SACS
-   MLCQ-related experiments/material
-   Figshare code-smell data
-   SoftDevl project material
-   RefactoringMiner-derived cases

The datasets differ in their definitions, labeling methods, granularity,
and smell coverage, so results are interpreted per dataset rather than
as a universal accuracy measurement.

### Selected Results

  --------------------------------------------------------------------------
  Experiment        |         Precision    |         Recall    |             F1
  ----------------- ------------------ ------------------ ------------------
  SmellyCode — file             85.71%             63.16%             72.73%
  scope, target                                           
  matching                                                

  MLCQ — revised                64.10%             83.33%             72.46%
  prompt, file                                            
  scope, target                                           
  matching                                                

  Refactoring                   30.00%             32.43%             31.17%
  recommendations —                                       
  revised                                                 
  target-based                                            
  result                                                  
  --------------------------------------------------------------------------

Performance varied across datasets and smell categories. The experiments
also highlighted challenges involving exact line localization,
heterogeneous dataset definitions, and refactorings that require broader
design context.

## Tech Stack

-   Java 21
-   Maven
-   GitHub API
-   GitHub Actions
-   Gemini API
-   Jackson
-   JUnit 5
-   JaCoCo
-   SLF4J

## Project Structure

    .
    |-- .github/workflows/       # CI and pull-request workflows
    |-- evaluation-dataset/      # Benchmark cases and labels
    |-- experiments/             # Experimental material
    |-- reports/                 # Evaluation/thesis-support reports
    |-- scripts/                 # Dataset and experiment utilities
    |
    |-- src/
    |   |-- main/java/dev/dimitra/bot/
    |   |   |-- Main.java
    |   |   |-- BotConfig.java
    |   |   |-- CodeSmellBotFacade.java
    |   |   |-- analysis/        # Prompting, analysis and validation
    |   |   |-- diff/            # Diff processing
    |   |   |-- eval/            # Local evaluation framework
    |   |   |-- github/          # GitHub integration
    |   |   |-- llm/             # LLM abstraction and Gemini client
    |   |   |-- model/           # Domain models
    |   |   `-- report/          # Report generation
    |   |
    |   `-- test/                # Automated tests
    |
    `-- pom.xml

## Getting Started

### Requirements

-   JDK 21
-   Maven or the included Maven wrapper
-   GitHub token for live Pull Request analysis
-   Gemini API key

### 1. Configure the environment

Use .env.example as the configuration template.

Important variables include:

    GITHUB_TOKEN=
    REPOSITORY=
    PR_NUMBER=

    POST_COMMENT=false
    ANALYSIS_SCOPE=diff

    LLM_PROVIDER=gemini
    GEMINI_API_KEY=
    GEMINI_MODEL=gemini-3.1-flash-lite

Additional variables control file limits, context retrieval, chunk
sizes, and LLM generation parameters.

Never commit API keys or GitHub tokens.

### 2. Build

Linux/macOS:

    ./mvnw clean package

Windows:

    .\mvnw.cmd clean package

The Maven build creates an executable shaded JAR under target/.

### 3. Run Tests

Linux/macOS:

    ./mvnw test

Windows:

    .\mvnw.cmd test

The test suite covers core components including changed-line parsing,
finding validation, smell analysis, dataset loading, evaluation scoring,
and report rendering.

### GitHub Actions

The repository contains workflows for automated pull-request analysis
and manual reruns.

At a high level, the workflow:

1.  retrieves the pull-request context and changed Java files;
2.  builds the analysis input;
3.  calls the configured LLM;
4.  parses the structured response;
5.  validates and anchors findings;
6.  generates the resulting review output.

Repository secrets should be used for provider credentials when running
the bot through GitHub Actions.

### Local Evaluation

After building the project, the evaluation framework can be run through
EvaluationMain.

Example dry run on Windows:

    java -cp target\code-smell-bot-0.1.0-SNAPSHOT.jar dev.dimitra.bot.eval.EvaluationMain --dataset evaluation-dataset\micro --scope file --dry-run

For an LLM-backed evaluation, configure the provider credentials first:

    $env:LLM_PROVIDER="gemini"
    $env:GEMINI_API_KEY="YOUR_API_KEY"

    java -cp target\code-smell-bot-0.1.0-SNAPSHOT.jar dev.dimitra.bot.eval.EvaluationMain --dataset evaluation-dataset\micro --scope file

Evaluation output can include:

    summary.json
    predictions.json
    report.md

See evaluation-dataset/README.md for additional dataset and evaluation
details.

## Design Decisions

Structured output over free-form reviews.
Machine-readable findings make validation, evaluation, and automated
reporting possible.

Shared analysis core.
The GitHub workflow and local evaluator reuse the same central analysis
components, reducing duplicated logic and making experiments more
representative of the actual bot.

Human-in-the-loop refactoring.
The system recommends refactorings but does not automatically modify
source code. Behavior-preserving automated refactoring would require
stronger compilation, testing, and semantic guarantees.

Multiple anchoring strategies.
Code smells often belong to methods or classes rather than one exact
source line, so the project supports both line-based and target-based
approaches.

## Limitations

-   LLM findings can be incorrect or incomplete.
-   Performance varies across datasets and smell categories.
-   Exact line localization can be unreliable.
-   Public datasets use different code-smell definitions and labeling
    strategies.
-   Some evaluation scenarios use synthetic diffs instead of real
    pull-request histories.
-   Refactoring recommendations may require broader architectural
    context.
-   Suggested refactorings are not automatically verified through
    compilation or behavioral-equivalence testing.
-   LLM execution introduces API latency, cost, and provider dependency.

## Future Work

Potential extensions include AST-based Java analysis, hybrid
LLM/static-analysis techniques, evaluation on larger collections of real
pull-request diffs, improved repository-level context retrieval,
stronger refactoring validation, and integration with deterministic
refactoring tools.

## Thesis

This project was developed as my Diploma Thesis in Computer Science &
Engineering at the University of Ioannina under the supervision of
Professor Apostolos Zarras.

The work investigates how an LLM can be integrated into a structured and
measurable software-engineering workflow for Java code-smell detection
and refactoring recommendation.

The main focus is not simply generating LLM suggestions, but making
those suggestions structured, validated, reproducible, and
experimentally evaluable.

## Author

Dimitra Christina Gkaravela
Computer Science & Engineering Graduate
University of Ioannina
