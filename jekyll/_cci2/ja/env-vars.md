---
layout: classic-docs
title: "環境変数の使い方"
short-title: "環境変数の使い方"
description: "CircleCI 2.0 で利用可能な環境変数一覧"
categories:
  - configuring-jobs
order: 40
---

このページでは CircleCI で利用可能な環境変数の使い方について、下記の内容に沿って解説しています。

- TOC
{:toc}

## Overview
{:.no_toc}

プロジェクトへのプライベート環境変数の追加は、CircleCI 上のプロジェクトごとの設定ページにある、**Environment Variables** で行えます。 環境変数にセットした実際の値は、ここでいったん設定すると、CircleCI 上では参照も編集もできません。 環境変数の値を変えたいときは、現在の環境変数を削除してから改めて別の値で作成し直してください。 環境変数は個別に追加したり、あるいは他のプロジェクトで定義している変数をインポートして追加できます。 また、プライベート環境変数は公開プロジェクトでも隠しておくことが可能です。これに関連する設定については[オープンソースプロジェクトのビルド方法]({{ site.baseurl }}/2.0/oss/)をご覧ください。 Use Contexts to further restrict access to environment variables from within the build, refer to the [Restricting a Context]({{ site.baseurl }}/2.0/contexts/#restricting-a-context) documentation.

### Secrets Masking
{:.no_toc}

Environment variables may hold project secrets or keys that perform crucial functions for your applications. For added security CircleCI performs secret masking on the build output, obscuring the `echo` or `print` output of environment variables and contexts.

The value of the environment variable will not be masked in the build output if:

- the value of the environment variable is less than 4 characaters
- the value of the environment variable is equal to one of `true`, `True`, `false` or `False`

**Note:** secret masking will only prevent the value of the environment variable from appearing in your build output. The value of the environment variable is still accessible to users [debugging builds with SSH]({{ site.baseurl }}/2.0/ssh-access-jobs).

### Environment Variable Usage Options
{:.no_toc}

CircleCI uses Bash, which follows the POSIX naming convention for environment variables. Valid characters include letters (uppercase and lowercase), digits, and the underscore. The first character of each environment variable must be a letter.

Secrets and private keys that are securely stored in the CircleCI app may be referenced with the variable in a `run` key, `environment` key, or a Workflows `context` key in your configuration. Environment variables are used according to a specific precedence order, as follows:

1. `run` ステップ内で指定している[シェルコマンド](#setting-an-environment-variable-in-a-shell-command)で宣言されたもの (例：`FOO=bar make install`)。
2. [`run` ステップ内](#setting-an-environment-variable-in-a-step) で `environment` キーを使って宣言されたもの。
3. [ jobs](#setting-an-environment-variable-in-a-job) 内において `environment` キーで定義したもの。
4. [コンテナ](#setting-an-environment-variable-in-a-container)において `environment` キーで定義したもの。
5. Context environment variables (assuming the user has access to the Context). See the [Contexts]({{ site.baseurl }}/2.0/contexts/) documentation for instructions.
6. プロジェクト設定ページで設定した[プロジェクトレベル環境変数](#setting-an-environment-variable-in-a-project)。
7. [CircleCI の定義済み環境変数](#built-in-environment-variables)で解説している特殊な環境変数。

Environment variables declared inside a shell command `run step`, for example `FOO=bar make install`, will override environment variables declared with the `environment` and `contexts` keys. Environment variables added on the Contexts page will take precedence over variables added on the Project Settings page. Finally, special CircleCI environment variables are loaded.

**Note**: Do not add secrets or keys inside the `.circleci/config.yml` file. The full text of `config.yml` is visible to developers with access to your project on CircleCI. Store secrets or keys in [project](#setting-an-environment-variable-in-a-project) or [context]({{ site.baseurl }}/2.0/contexts/) settings in the CircleCI app.

For more information, see the [Encryption section]({{ site.baseurl }}/2.0/security/#encryption) of the "Security" document.

Running scripts within configuration may expose secret environment variables. See the [Using Shell Scripts]({{ site.baseurl }}/2.0/using-shell-scripts/#shell-script-best-practices) document for best practices for secure scripts.

### Example Configuration of Environment Variables

Consider the example `config.yml` below:

```yaml
version: 2  # use CircleCI 2.0
jobs: # basic units of work in a run
  build: # runs not using Workflows must have a `build` job as entry point
    docker: # run the steps with Docker
      # CircleCI node images available at: https://hub.docker.com/r/circleci/node/
      - image: circleci/node:10.0-browsers
    steps: # steps that comprise the `build` job
      - checkout # check out source code to working directory
      # Run a step to setup an environment variable.
      - run: 
          name: "Setup custom environment variables"
          command: |
            echo 'export MY_ENV_VAR="FOO"' >> $BASH_ENV # Redirect MY_ENV_VAR into $BASH_ENV
      # Run a step to print what branch our code base is on.
      - run: # test what branch we're on.
          name: "What branch am I on?"
          command: echo ${CIRCLE_BRANCH}
      # Run another step, the same as above; note that you can
      # invoke environment variable without curly braces.
      # prints: XXXXXXX
      - run:
          name: "What branch am I on now?"
          command: echo $CIRCLE_BRANCH # prints: XXXXXXX
      - run:
          name: "What was my custom environment variable?"
          command: echo ${MY_ENV_VAR}  # prints: XXXXXXX
```

The above `config.yml` demonstrates the following:

- Setting custom environment variables
- Reading a built-in environment variable that CircleCI provides (`CIRCLE_BRANCH`)
- How variables are used (or interpolated) in your `config.yml`
- Masking of printed environment variables (secrets masking)

When the above config runs, the output looks like this:

![Env Vars Interpolation Example]({{site.baseurl}}/assets/img/docs/env-vars-interpolation-example.png)

You may have noticed that there are two similar steps in the above image and config - "What branch am I on?". These steps illustrate two different methods to read environment variables. Note that both `${VAR}` and `$VAR` syntaxes are supported. You can read more about shell parameter expansion in the [Bash documentation](https://www.gnu.org/software/bash/manual/bashref.html#Shell-Parameter-Expansion).

### Using `BASH_ENV` to Set Environment Variables
{:.no_toc}

CircleCI does not support interpolation when setting environment variables. All defined values are treated literally. This can cause issues when defining `working_directory`, modifying `PATH`, and sharing variables across multiple `run` steps.

In the example below, `$ORGNAME` and `$REPONAME` will not be interpolated.

```yaml
working_directory: /go/src/github.com/$ORGNAME/$REPONAME
```

There are several workarounds to interpolate values into your config.

If you are using **CircleCI 2.1**, you can reuse pieces of config across your `config.yml`. By using the `parameters` declaration, you can interpolate (or, "pass values") into reusable `commands` `jobs` and `executors`. Consider reading the documentation on [using the parameters declaration]({{ site.baseurl }}/2.0/reusing-config/#using-the-parameters-declaration).

Another possible method to interpolate values into your config is to use a `run` step to export environment variables to `BASH_ENV`, as shown below.

```yaml
steps:
  - run:
      name: Setup Environment Variables
      command: |
        echo "export PATH=$GOPATH/bin:$PATH" >> $BASH_ENV
        echo "export GIT_SHA1=$CIRCLE_SHA1" >> $BASH_ENV
```

In every step, CircleCI uses `bash` to source `BASH_ENV`. This means that `BASH_ENV` is automatically loaded and run, allowing you to use interpolation and share environment variables across `run` steps.

**Note:** The `$BASH_ENV` workaround only works with `bash`. Other shells probably won't work.

### Alpine Linux

An image that's based on [Alpine Linux](https://alpinelinux.org/) (like [docker](https://hub.docker.com/_/docker)), uses the `ash` shell.

To use environment variables with `ash`, just add these 2 parameters to your job.

```yaml
jobs:
  build:    

    shell: /bin/sh -leo pipefail
    environment:

      - BASH_ENV: /etc/profile
```

## シェルコマンドで環境変数を設定する

While CircleCI does not support interpolation when setting environment variables, it is possible to set variables for the current shell by [using `BASH_ENV`](#using-bash_env-to-set-environment-variables). This is useful for both modifying your `PATH` and setting environment variables that reference other variables.

```yaml
version: 2
jobs:
  build:
    docker:
      - image: smaant/lein-flyway:2.7.1-4.0.3
    steps:
      - run:
          name: Update PATH and Define Environment Variable at Runtime
          command: |
            echo 'export PATH=/path/to/foo/bin:$PATH' >> $BASH_ENV
            echo 'export VERY_IMPORTANT=$(cat important_value)' >> $BASH_ENV
            source $BASH_ENV
```

**Note**: Depending on your shell, you may have to append the new variable to a shell startup file like `~/.tcshrc` or `~/.zshrc`.

For more information, refer to your shell's documentation on setting environment variables.

## ステップ内で環境変数を設定する

To set an environment variable in a step, use the [`environment` key]({{ site.baseurl }}/2.0/configuration-reference/#run).

```yaml
version: 2
jobs:
  build:
    docker:
      - image: smaant/lein-flyway:2.7.1-4.0.3
    steps:
      - checkout
      - run:
          name: Run migrations
          command: sql/docker-entrypoint.sh sql
          # Environment variable for a single command shell
          environment:
            DATABASE_URL: postgres://conductor:@localhost:5432/conductor_test
```

**Note:** Since every `run` step is a new shell, environment variables are not shared across steps. If you need an environment variable to be accessible in more than one step, export the value [using `BASH_ENV`](#using-bash_env-to-set-environment-variables).

## ジョブ内で環境変数を設定する

To set an environment variable in a job, use the [`environment` key]({{ site.baseurl }}/2.0/configuration-reference/#job_name).

```yaml
version: 2
jobs:
  build:
    docker:
      - image: buildpack-deps:trusty
    environment:
      FOO: bar
```

## コンテナ内で環境変数を設定する

To set an environment variable in a container, use the [`environment` key]({{ site.baseurl }}/2.0/configuration-reference/#docker--machine--macos--windows-executor).

```yaml
version: 2
jobs:
  build:
    docker:
      - image: smaant/lein-flyway:2.7.1-4.0.3
      - image: circleci/postgres:9.6-jessie
      # environment variables for all commands executed in the primary container
        environment:
          POSTGRES_USER: conductor
          POSTGRES_DB: conductor_test
```

The following example shows separate environment variable settings for the primary container image (listed first) and the secondary or service container image.

```yaml
version: 2
jobs:
  build:
    docker:
      - image: circleci/python:3.6.2-jessie
       # environment variables for all commands executed in the primary container
        environment:
          FLASK_CONFIG: testing
          TEST_DATABASE_URL: postgresql://ubuntu@localhost/circle_test?sslmode=disable
      - image: circleci/postgres:9.6
```

## コンテキスト内で環境変数を設定する

Creating a context allows you to share environment variables across multiple projects. To set an environment variables in a context, see the [Contexts documentation]({{ site.baseurl }}/2.0/contexts/).

## プロジェクト内で環境変数を設定する

1. In the CircleCI application, go to your project's settings by clicking the gear icon next to your project.

2. In the **Build Settings** section, click on **Environment Variables**.

3. Import variables from another project by clicking the **Import Variable(s)** button. Add new variables by clicking the **Add Variable** button. (**注：****Import Variables** ボタンは、プライベートクラウドやデータセンターにインストールした CircleCI では現在利用できません。)

4. `.circleci/config.yml` に新しい環境変数を追加します。 For an example, see the [Heroku deploy walkthrough]({{ site.baseurl }}/2.0/deployment-integrations/#heroku).

Once created, environment variables are hidden and uneditable in the application. Changing an environment variable is only possible by deleting and recreating it.

### Encoding Multi-Line Environment Variables
{:.no_toc}

If you are having difficulty adding a multiline environment variable, use `base64` to encode it.

```bash
$ echo "foobar" | base64 --wrap=0
Zm9vYmFyCg==
```

Store the resulting value in a CircleCI environment variable.

```bash
$ echo $MYVAR
Zm9vYmFyCg==
```

Decode the variable in any commands that use the variable.

```bash
$ echo $MYVAR | base64 --decode | docker login -u my_docker_user --password-stdin
Login Succeeded
```

**Note:** Not all command-line programs take credentials in the same way that `docker` does.

## API に環境変数を代入する方法

Build parameters are environment variables, therefore their names have to meet the following restrictions:

- They must contain only ASCII letters, digits and the underscore character.
- They must not begin with a number.
- They must contain at least one character.

Aside from the usual constraints for environment variables there are no restrictions on the values themselves and are treated as simple strings. The order that build parameters are loaded in is **not** guaranteed so avoid interpolating one build parameter into another. It is best practice to set build parameters as an unordered list of independent environment variables.

**IMPORTANT** Build parameters are not treated as sensitive data and must not be used by customers for sensitive values (secrets). You can find this sensitive information in [Project Settings](https://circleci.com/docs/2.0/settings/) and [Contexts](https://circleci.com/docs/2.0/glossary/#context).

For example, when you pass the parameters:

    {
      "build_parameters": {
        "foo": "bar",
        "baz": 5,
        "qux": {"quux": 1},
        "list": ["a", "list", "of", "strings"]
      }
    }
    

Your build will see the environment variables:

    export foo="bar"
    export baz="5"
    export qux="{\"quux\": 1}"
    export list="[\"a\", \"list\", \"of\", \"strings\"]"
    

Build parameters are exported as environment variables inside each job's containers and can be used by scripts/programs and commands in `config.yml`. The injected environment variables may be used to influence the steps that are run during the job. It is important to note that injected environment variables will not override values defined in `config.yml` nor in the project settings.

You might want to inject environment variables with the `build_parameters` key to enable your functional tests to build against different targets on each run. For example, a run with a deploy step to a staging environment that requires functional testing against different hosts. It is possible to include `build_parameters` by sending a JSON body with `Content-type: application/json` as in the following example that uses `bash` and `curl` (though you may also use an HTTP library in your language of choice).

    {
      "build_parameters": {
        "param1": "value1",
        "param2": 500
      }
    }
    

For example using `curl`

    curl \
      --header "Content-Type: application/json" \
      --data '{"build_parameters": {"param1": "value1", "param2": 500}}' \
      --request POST \
      https://circleci.com/api/v1.1/project/github/circleci/mongofinil/tree/master?circle-token=$CIRCLE_TOKEN
    

In the above example, `$CIRCLE_TOKEN` is a [personal API token]({{ site.baseurl }}/2.0/managing-api-tokens/#creating-a-personal-api-token).

The build will see the environment variables:

    export param1="value1"
    export param2="500"
    

Start a run with the POST API call, see the [new build](https://circleci.com/docs/api/#trigger-a-new-build-with-a-branch) section of the API documentation for details. A POST with an empty body will start a new run of the named branch.

## 定義済み環境変数

The following environment variables are exported in each build and can be used for more complex testing or deployment.

**Note:** You cannot use a built-in environment variable to define another environment variable. Instead, you must use a `run` step to export the new environment variables using `BASH_ENV`.

For more details, see [Setting an Environment Variable in a Shell Command](#setting-an-environment-variable-in-a-shell-command).

| Variable                    | Type    | Value                                                                                                                                                                                |
| --------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `CI`                        | Boolean | `true` (represents whether the current environment is a CI environment)                                                                                                              |
| `CI_PULL_REQUEST`           | String  | Deprecated version of `CIRCLE_PULL_REQUEST`. Kept for backward compatibility with CircleCI 1.0.                                                                                      |
| `CI_PULL_REQUESTS`          | List    | Deprecated version of `CIRCLE_PULL_REQUESTS`. Kept for backward compatibility with CircleCI 1.0.                                                                                     |
| `CIRCLE_BRANCH`             | String  | The name of the Git branch currently being built.                                                                                                                                    |
| `CIRCLE_BUILD_NUM`          | Integer | The number of the CircleCI build.                                                                                                                                                    |
| `CIRCLE_BUILD_URL`          | String  | The URL for the current build.                                                                                                                                                       |
| `CIRCLE_COMPARE_URL`        | String  | The GitHub or Bitbucket URL to compare commits of a build. Available in config v2 and below. For v2.1 we will introduce "pipeline values" as an alternative.                         |
| `CIRCLE_INTERNAL_TASK_DATA` | String  | The directory where test timing data is saved.                                                                                                                                       |
| `CIRCLE_JOB`                | String  | The name of the current job.                                                                                                                                                         |
| `CIRCLE_NODE_INDEX`         | Integer | The index of the specific build instance. A value between 0 and (`CIRCLECI_NODE_TOTAL` - 1)                                                                                          |
| `CIRCLE_NODE_TOTAL`         | Integer | The number of total build instances.                                                                                                                                                 |
| `CIRCLE_PR_NUMBER`          | Integer | The number of the associated GitHub or Bitbucket pull request. フォークしたプルリクエストのみで使用可能です。                                                                                               |
| `CIRCLE_PR_REPONAME`        | String  | The name of the GitHub or Bitbucket repository where the pull request was created. フォークしたプルリクエストのみで使用可能です。                                                                           |
| `CIRCLE_PR_USERNAME`        | String  | The GitHub or Bitbucket username of the user who created the pull request. フォークしたプルリクエストのみで使用可能です。                                                                                   |
| `CIRCLE_PREVIOUS_BUILD_NUM` | Integer | The number of previous builds on the current branch.                                                                                                                                 |
| `CIRCLE_PROJECT_REPONAME`   | String  | The name of the repository of the current project.                                                                                                                                   |
| `CIRCLE_PROJECT_USERNAME`   | String  | The GitHub or Bitbucket username of the current project.                                                                                                                             |
| `CIRCLE_PULL_REQUEST`       | String  | The URL of the associated pull request. ひも付けられたプルリクエストが複数ある時は、そのうちの 1 つがランダムで選ばれます。                                                                                                  |
| `CIRCLE_PULL_REQUESTS`      | List    | Comma-separated list of URLs of the current build's associated pull requests.                                                                                                        |
| `CIRCLE_REPOSITORY_URL`     | String  | The URL of your GitHub or Bitbucket repository.                                                                                                                                      |
| `CIRCLE_SHA1`               | String  | The SHA1 hash of the last commit of the current build.                                                                                                                               |
| `CIRCLE_TAG`                | String  | The name of the git tag, if the current build is tagged. For more information, see the [Git Tag Job Execution]({{ site.baseurl }}/2.0/workflows/#executing-workflows-for-a-git-tag). |
| `CIRCLE_USERNAME`           | String  | The GitHub or Bitbucket username of the user who triggered the build.                                                                                                                |
| `CIRCLE_WORKFLOW_ID`        | String  | A unique identifier for the workflow instance of the current job. この ID は Workflow インスタンス内のすべてのジョブで同一となります。                                                                          |
| `CIRCLE_WORKING_DIRECTORY`  | String  | The value of the `working_directory` key of the current job.                                                                                                                         |
| `CIRCLECI`                  | Boolean | `true` (represents whether the current environment is a CircleCI environment)                                                                                                        |
| `HOME`                      | String  | Your home directory.                                                                                                                                                                 |
{:class="table table-striped"}

## See Also
{:.no_toc}

[Contexts]({{ site.baseurl }}/2.0/contexts/) [Keep environment variables private with secret masking](https://circleci.com/blog/keep-environment-variables-private-with-secret-masking/)