# testilo
Utilities for Testaro

## Introduction

The Testilo package contains utilities that facilitate the use of the [Testaro](https://www.npmjs.com/package/testaro) package.

Testaro performs digital accessibility tests on web artifacts and creates reports in JSON format. The utilities in Testilo fall into two categories:
- Job preparation
- Report enhancement

Because Testilo supports Testaro, this `README` file presumes that you have access to the Testaro `README` file and therefore does not repeat information provided there.

## Dependencies

The `dotenv` dependency lets you set environment variables in an untracked `.env` file. This prevents secret data, such as passwords, from being shared as part of this package.

When Testilo is a dependency of another application, the `.env` file is not imported, because it is not tracked, so all needed environment variables must be provided by the importing application.

## Architecture

Testilo is written in Node.js. Commands are given to Testilo in a command-line (terminal) interface or programmatically.

Shared routines are _procs_ and are located in the `procs` directory.

Testilo can be installed wherever Node.js (version 14 or later) is installed. This can be a server or the same workstation on which Testaro is installed.

The reason for Testilo being an independent package, rather than part of Testaro, is that Testilo can be installed on any host, while Testaro can run successfully only on a Windows or Macintosh workstation (and perhaps on some workstations with Ubuntu operating systems). Testaro runs tests similar to those that a human accessibility tester would run, using whatever browsers, input devices, system settings, simulated and attached devices, and assistive technologies tests may require. Thus, Testaro is limited to functionalities that require workstation attributes. For maximum flexibility in the management of Testaro jobs, all other functionalities are located outside of Testaro. You could have software such as Testilo running on a server, communicating with multiple workstations running Testaro. The workstations could receive job orders from the server and return job results to the server for further processing.

## Job preparation

### Introduction

Testaro executes _jobs_. In a job, Testaro performs _acts_ (tests and other operations) on _targets_ (typically, web pages). The Testaro `README.md` file specifies the requirements for a job.

You can create a job for Testaro directly, without using Testilo.

Testilo can, however, make job preparation more efficient in two scenarios:
- A common use case is to define a battery of tests and to have those tests performed on multiple targets. In that case, you need to create multiple jobs, one per page. The jobs are identical, except for the target-specific acts (including navigating to targets). If you tell Testilo about the tests and the targets, Testilo can create such collections of jobs for you.
- Some tests operate on a copy of a target and modify that copy. Usually, one wants test isolation: The results of a test do not depend on any previously performed tests. To ensure test isolation, a job containing such target-modifying tests must follow them with acts that restore the target to its desired pre-test state. Testilo can insert the required target-restoring acts into each job after target-modifying tests.

The `merge` module performs these services.

### Procedure

To use the `merge` module, you need to create a _script_ and a _batch_:
- The script is an object that specifies a job, except for the targets.
- The batch is an object that specifies target-specific acts for targets.

The script contains an array of acts, with placeholders. For each target in the batch, the `merge` module creates a job, in which the placeholders are replaced with the acts specific to that target in the batch.

### Example

Here is an example illustrating how merging works.

#### Script

Suppose you have created this script:

```javaScript
{
  id: 'ts25',
  what: 'Motion, Axe, and bulk',
  strict: true,
  timeLimit: 60,
  acts: [
    {
      type: 'placeholder',
      which: 'main',
      launch: 'webkit'
    },
    {
      type: 'test',
      which: 'motion',
      what: 'spontaneous change of content; requires webkit',
      delay: 2500,
      interval: 2500,
      count: 5
    },
    {
      type: 'placeholder',
      which: 'main',
      launch: 'chromium'
    },
    {
      type: 'test',
      which: 'axe',
      withItems: true,
      rules: [],
      what: 'Axe core, all rules, with details'
    },
    {
      type: 'test',
      which: 'bulk',
      what: 'count of visible elements'
    }
  ]
}
```

The `acts` array of this script contains three acts of type `test` and two acts of type `placeholder`.

#### Batch

Suppose you have also created this batch:

```javaScript
{
  id: 'weborgs',
  what: 'Web standards organizations',
  targets: [
    {
      id: 'mozilla',
      what: 'Mozilla Foundation',
      acts: {
        main: [
          {
            type: 'launch'
          },
          {
            type: 'url',
            which: 'https://foundation.mozilla.org/en/',
            what: 'Mozilla Foundation'
          }
        ]
      }
    },
    {
      id: 'w3c',
      what: 'World Wide Web Consortium',
      acts: {
        main: [
          {
            type: 'launch'
          },
          {
            type: 'url',
            which: 'https://www.w3.org/',
            what: 'World Wide Web Consortium'
          }
        ]
      }
    }
  ]
}
```

This batch defines two targets.

#### Isolation option

The `merge` module offers an isolation option. If you exercise it, the `merge` module will act as if the latest placeholder were again inserted after each target-modifying test, except where that test is the last act or the next act after it is a placeholder.

#### Output

Suppose you ask for a merger of the above batch and script, **without** the isolation option. Then the first job produced by `merge` could be:

```javaScript
{
  id: '7izc1-ts25-mozilla',
  what: 'Motion, Axe, and bulk',
  strict: true,
  timeLimit: 60,
  acts: [
    {
      type: 'launch',
      which: 'webkit'
    },
    {
      type: 'url',
      which: 'https://foundation.mozilla.org/en/',
      what: 'Mozilla Foundation'
    }
    {
      type: 'test',
      which: 'motion',
      what: 'spontaneous change of content; requires webkit',
      delay: 2500,
      interval: 2500,
      count: 5
    },
    {
      type: 'launch',
      which: 'chromium'
    },
    {
      type: 'url',
      which: 'https://foundation.mozilla.org/en/',
      what: 'Mozilla Foundation'
    }
    {
      type: 'test',
      which: 'axe',
      withItems: true,
      rules: [],
      what: 'Axe core, all rules, with details'
    },
    {
      type: 'test',
      which: 'bulk',
      what: 'count of visible elements'
    }
  ],
  sources: {
    script: 'ts25',
    batch: 'weborgs',
    target: {
      id: 'mozilla',
      what: 'Mozilla Foundation'
    },
    requester: 'you@yourdomain.tld'
  },
  creationTime: '2023-11-20T15:50:27',
  timeStamp: 'bizc1'
}
```

Most of the properties of this job object come from your script and your batch. The acts of type `placeholder` in the script have been replaced with the specified (i.e. `main`) array of acts of the first target of the batch. In this case, that target has only one array of acts, and that array contains two acts, but a target could have multiple act arrays with distinct names, and an act array could include any number of acts or be empty.

If the named array of acts includes an act of type `launch`, that act gets a `which` property, identical to the value of the `launch` property of the `placeholder` object. In this way, a placeholder can dictate which browser type is to be launched.

The `merge` module has added other properties to the job:
- a creation time
- a timestamp, derived from the creation time
- a job ID, composed of the timestamp, the script ID, and the target ID
- a `sources` property, identifying the script, the batch, the target within the batch, and an email address obtained from the environment variable `process.env.REQUESTER`.

This job is ready to be executed by Testaro.

If, however, you requested a merger **with** isolation, then `merge` would act as if another instance of

    ```javaScript
    {
      type: 'placeholder',
      which: 'main',
      launch: 'chromium'
    },
    ```

were located in the script after the `axe` test, because the `axe` test is target-modifying and could therefore change the result of the `bulk` test that follows it. So, additional acts of type `launch` and `url` would appear after the `axe` test in the job.

### Invocation

There are two ways to use the `merge` module.

#### By a module

A module can invoke `merge` in this way:

```javaScript
const {merge} = require('testilo/merge');
const jobs = merge(script, batch, requester, true);
```

This invocation references `script`, `batch`, and `requester` variables that the module must have already defined. The `script` and `batch` variables are a script object and a batch object, respectively. The `requester` variable is an email address. The fourth argument is a boolean, specifying whether to perform isolation; a missing fourth argument is equivalent to `false`. The `merge()` function of the `merge` module generates jobs and returns them in an array. The invoking module can further dispose of the jobs as needed.

#### By a user

A user can invoke `merge` in this way:

- Create a script and save it as a JSON file in the `scripts` subdirectory of the `process.env.SPECDIR` directory.
- Create a batch and save it as a JSON file in the `batches` subdirectory of the `process.env.SPECDIR` directory.
- In the Testilo project directory, execute one of these statements:
    - `node call merge s b e true`
    - `node call merge s b e false`
    - `node call merge s b e`

In these statements, replace `s` and `b` with the base names of the script and batch files, respectively. For example, if the script file is named `ts25.json`, then replace `x` with `ts25`. Replace `e` with an email address, or with an empty string if the environment variable `process.env.REQUESTER` exists and you want to use it.

The first statement will cause a merger with isolation.
The second and third statements will cause a merger without isolation.

The `call` module will retrieve the named script and batch from their respective directories.
The `merge` module will create an array of jobs.
The `call` module will save the jobs in the `todo` subdirectory of the `process.env.JOBDIR` directory.

### Validation

To test the `merge` module, in the project directory you can execute the statement `node validation/merge/validate`. All logging statements should begin with “Success” and none should begin with “ERROR”.

### Examples

There are script and batch examples in the `specs` directory.

The script example (with ID `ts21`) performs all of the tests available in Testaro, including the tests of 9 dependent tools. It contains 2 `placeholder` acts, one that specifies the Webkit browser for any `launch` act and one that specifies the Chromium browser for any `launch` act.

The batch example (with ID `orgs`) specifies 2 targets.
- One of them has the URL `https://example.com` and requires each placeholder to be replaced with 2 acts: a `launch` act and a `url` act.
- The other has the URL `https://www.w3.org/WAI/ARIA/apg/example-index/accordion/accordion` and requires each placeholder to be replaced with 3 acts: a `launch` act, a `url` act, and a `button` act.

## Report scoring

### Introduction

Testaro executes jobs and produces reports of test results. A report is identical to a job (see the example above), except that:
- The acts contain additional data recorded by Testaro to describe the results of the performance of the acts. Acts of type `test` have additional data describing test results (successes, failures, and details).
- Testaro also adds a `jobData` property, describing information not specific to any particular act.

Thus, a report produced by Testaro contains these properties:
- `id`
- `what`
- `strict`
- `timeLimit`
- `acts`
- `sources`
- `creationTime`
- `timeStamp`
- `jobData`

Testilo can add scores to a report. In this way, a report can not only detail successes and failures of individual tests but also assign scores to those results and combine the partial scores into total scores. The scores are contained in a new `score` property that Testilo adds to a report.

The `score` module scores a report. Its `score()` function takes two arguments:
- a scoring function
- an array of report objects

A scoring function defines scoring rules. The Testilo package contains a `procs/score` directory, in which there are modules that export scoring functions. You can use one of those scoring functions, or you can create your own.

### Invocation

There are two ways to invoke the `score` module.

#### By a module

A module can invoke `score` in this way:

```javaScript
const {score} = require('testilo/score');
const {scorer} = require('testilo/procs/score/sp25a');
const scoredReports = score(scorer, rawReports);
```

The first argument to `score()` is a scoring function. In this example, it has been obtained from a JSON file in the Testilo package, but it could be a custom function. The second argument to `score()` is an array of report objects.

#### By a user

A user can invoke `score` in this way:

```bash
node call score tsp25a
node call score tsp25a 75
```

When a user invokes `score` in this example, the `call` module:
- gets the scoring module `tsp25a` from its JSON file `tsp25a.json` in the `score` subdirectory of the `process.env.FUNCTIONDIR` directory
- gets the reports from the `raw` subdirectory of the `process.env.REPORTDIR` directory
- writes the scored reports to the `scored` subdirectory of the `process.env.REPORTDIR` directory

The optional third argument to call (`75` in this example) is a report selector. Without the argument, `call` gets all the reports in the `raw` subdirectory. With the argument, `call` gets only those reports whose names begin with the argument string.

### Validation

To test the `score` module, in the project directory you can execute the statement `node validation/score/validate`. All logging statements should begin with “Success” and none should begin with “ERROR”.

### Comprehensive example

The module `procs/score/tsp21.js` is an example of a comprehensive score proc. It is designed to score reports produced by Testaro from jobs that Testilo creates from the `ts21` script. That script causes Testaro to perform 1351 tests belonging to Testaro and nine other tools integrated into Testaro as dependencies.

The `tsp21` score proc combines the results of those tests into a single score for each report. The `score` property that the proc adds to the report shows how the individual test results are aggregated into a score.

The `tsp21` score proc imports the `procs/score/tic21.js` module, which classifies all the tests into 255 “issues”. Any two or more tests classified into the same issue are deemed to be approximately equivalent in intent. Using this classification, the score proc tabulates the test results by issue, so that all instances of the issue, regardless of which tool discovered the instances, are reported together.

The issue classification in `tic21` assigns weights to the issues to reflect their estimated severities,and the scoring algorithm of `tsp21` adopts a compromise between issue-based and tool-based summation. Thus, if 3 tools discover 20 instances of an issue, the score increases (i.e. becomes worse) by more than it would if only 1 tool had discovered them, but less than 3 times as much. The motivations are that discovery of issue instances by multiple tools increases confidence that the issues are real, and passing accessibility tests is per se beneficial because it decreases risks associated with alleged inaccessibility.

The `tsp21` score proc does **not** attempt to identify instances of issues. Thus, if tool A discovers 4 instances and tool B discovers 7 instances of some issue, `tsp21` does not attempt to say which of the 4 discovered by A are a subset of the 7 discovered by B. Instead,`tsp21` naïvely acts as if the 4 are all a subset of the 7, even though in reality the two sets might be entirely disjoint.

In the `tsp21` scoring algorithm, the failure of a tool or of a Testaro test to run on a page increases the score of the page, and messages logged by the browser while tests are being performed on a page, especially error messages, also increase the score.

## Report digesting

### Introduction

Reports created by Testaro, both originally and after they are scored, are JavaScript objects. When represented as JSON, they are human-readable, but not human-friendly. They are basically designed for machine tractability. Testilo can _digest_ a scored report, converting it to a human-oriented HTML document, or _digest_.

The `digest` module digests a scored report. Its `digest()` function takes three arguments:
- a digest template
- a digesting function
- an array of report objects

The digest template is an HTML document containing placeholders. A copy of the template, with its placeholders replaced by texts, becomes the digest. The digesting function defines the rules for replacing the placeholders with texts. The Testilo package contains a `procs/digest` directory, in which there are subdirectories containing pairs of templates and modules that export digesting functions. You can use one of those pairs, or you can create your own.

### Invocation

There are two ways to use the `digest` module.

#### By a module

A module can invoke `digest` in this way:

```javaScript
const {digest} = require('testilo/digest');
const digesterDir = `${process.env.FUNCTIONDIR}/digest/dp25a`;
fs.readFile(`${digesterDir}/index.html`)
.then(template => {
  const {digester} = require(`${digesterDir}/index`);
  const digestedReports = digest(template, digester, scoredReports);
});
```

The first two arguments to `digest()` are a digest template and a digesting function. In this example, they have been obtained from files in the Testilo package, but they could be custom-made. The third argument to `digest()` is an array of report objects.

#### By a user

A user can invoke `digest` in this way:

```bash
node call digest dp25a
node call digest dp25a 75
```

When a user invokes `digest` in this example, the `call` module:
- gets the template and the digesting module from subdirectory `dp25a` in the `digest` subdirectory of the `process.env.FUNCTIONDIR` directory
- gets the reports from the `scored` subdirectory of the `process.env.REPORTDIR` directory
- writes the digested reports to the `digested` subdirectory of the `process.env.REPORTDIR` directory

The optional third argument to call (`75` in this example) is a report selector. Without the argument, `call` gets all the reports in the `scored` subdirectory of the `process.env.REPORTDIR` directory. With the argument, `call` gets only those reports whose names begin with the argument string.

The digests created by `digest` are HTML files, and they expect a `style.css` file to exist in their directory. The `reports/digested/style.css` file in Testilo is an appropriate stylesheet to be copied into the directory where digested reports are written.

### Validation

To test the `digest` module, in the project directory you can execute the statement `node validation/digest/validate`. All logging statements should begin with “Success” and none should begin with “ERROR”.

### Report comparison

If you use Testilo to perform a battery of tests on multiple targets, you may want a single report that compares the total scores received by the targets. Testilo can produce such a _comparative report_.

The `compare` module compares the scores in a collection of scored reports. Its `compare()` function takes three arguments:
- a comparison template
- a comparison function
- an array of scored reports

The comparison template is an HTML document containing placeholders. A copy of the template, with its placeholders replaced by texts, becomes the comparative report. The comparison function defines the rules for replacing the placeholders with texts. The Testilo package contains a `procs/compare` directory, in which there are subdirectories containing pairs of templates and modules that export comparison functions. You can use one of those pairs, or you can create your own.

### Invocation

There are two ways to use the `compare` module.

#### By a module

A module can invoke `compare` in this way:

```javaScript
const fs = require('fs/promises);
const {compare} = require('testilo/compare');
const comparerDir = `${process.env.FUNCTIONDIR}/compare/cp25a`;
fs.readFile(`${comparerDir}/index.html`)
.then(template => {
  const {comparer} = require(`${comparerDir}/index`);
  const comparativeReports = compare(template, comparer, scoredReports);
});
```

The first two arguments to `compare()` are a template and a comparison function. In this example, they have been obtained from files in the Testilo package, but they could be custom-made. The third argument to `compare()` is an array of report objects.

#### By a user

A user can invoke `compare` in this way:

```bash
node call compare cp25a legislators
```

When a user invokes `compare` in this example, the `call` module:
- gets the template and the comparison module from subdirectory `cp25a` of the subdirectory `compare` in the `process.env.FUNCTIONDIR` directory
- gets all the reports in the `scored` subdirectory of the `process.env.REPORTDIR` directory
- writes the comparative report as an HTML file named `legislators.html` to the `comparative` subdirectory of the `process.env.REPORTDIR` directory

The comparative reports created by `compare` are HTML files, and they expect a `style.css` file to exist in their directory. The `reports/comparative/style.css` file in Testilo is an appropriate stylesheet to be copied into the directory where comparative reports are written.

### Validation

To test the `compare` module, in the project directory you can execute the statement `node validation/compare/validate`. All logging statements should begin with “Success” and none should begin with “ERROR”.
