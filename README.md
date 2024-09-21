# This is a fork!!!

This is a fork of https://github.com/kzkn/simplecov-resultset-diff-action. It is not 
([Pablo](https://github.com/pupeno))'s intention to take over `simplecov-resultset-diff-action`, but he needed to 
upgrade dependencies and possible make other changes for this to work and all those changes are being maintained here,
in https://github.com/pupeno/simplecov-resultset-diff-action. Some of the changes were packaged in a PR and if PR's
start to be merged, Pablo is happy to prepare more.

If you are considering using this fork, you can see the diff with kzkn's on 
https://github.com/kzkn/simplecov-resultset-diff-action/compare/main...pupeno:simplecov-resultset-diff-action:main

# SimpleCov Resultset Diff

Creates a comment inside your Pull-Request with the difference between two SimpleCov resultset files.

![Comment demo](./docs/splash.png)

## Usage

To use this Github action, in your steps you may have:

```yaml
uses: pupeno/simplecov-resultset-diff-action@v1.2
with:
  base-resultset-path: '/path/to/my/.resultset.json'
  head-resultset-path: '/path/to/my/.resultset.json'
  token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Inputs          | Required | Default | Description                                                                                   |
|-----------------|----------|---------|-----------------------------------------------------------------------------------------------|
| base-stats-path | true     |         | Path to the SimpleCov generated ".resultset.json" file from the base branch.                  |
| head-stats-path | true     |         | Path to the SimpleCov generated "resultset.json" file from the head branch.                   |
| token           | true     |         | Github token so the package can publish a comment in the pull-request when the diff is ready. |

## Usage example

If you want to compare the coverage difference between your base branch and your pull-request head branch.

You'll need to run your test and collect coverage for the head branch with something like this (you may need to set up
a database or do other things, this is just an example):

```yaml
name: Pull Request
on: pull_request
jobs:
  test-head:
    name: Test head branch
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Ruby and gems
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Set up database schema
        run: bin/rails db:migrate

      - name: Precompile assets
        run: bundle exec rake assets:precompile

      - name: Run tests
        run: bundle exec rspec
```

Then we will use the Github Actions feature called "[artifacts](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts)" to store the generated `coverage/.resultset.json` file.

```yaml
      - name: Upload Coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: head-result
          path: coverage/.resultset.json
          if-no-files-found: error
          include-hidden-files: true
```

Now you can do the exact same thing, but for the base branch. Note the checkout step!

```yaml
  test-base:
    name: Test base branch
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ## Here we do not check out the current branch, but we check out the base branch.
          ref: ${{ github.base_ref }}

      - id: base-ref-commit
        run: echo "revision=`git rev-parse HEAD`" >> $GITHUB_ENV

      - name: Install Ruby and gems
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Set up database schema
        run: bin/rails db:migrate

      - name: Precompile assets
        run: bundle exec rake assets:precompile

      - name: Run tests
        run: bundle exec rspec

      - name: Upload Coverage
        uses: actions/upload-artifact@v4
        with:
          name: base-result
          path: coverage/.resultset.json
          if-no-files-found: error
          include-hidden-files: true
```

Now, in a new job we can retrieve both of our saved resultsets from the artifacts and use this action to compare them.

```yaml
  compare-coverage:
    name: Compare Coverage
    runs-on: ubuntu-latest
    needs: [test-head, test-base]

    steps:
      - name: Download Base Coverage
        uses: actions/download-artifact@v4
        with:
          name: base-result
          path: ./base-result/

      - name: Download Current Coverage
        uses: actions/download-artifact@v4
        with:
          name: head-result
          path: ./head-result/

      - run: find .

      - uses: pupeno/simplecov-resultset-diff-action@v1.2
        with:
          base-resultset-path: ./base-result/.resultset.json
          head-resultset-path: ./head-result/.resultset.json
          token: ${{ secrets.GITHUB_TOKEN }}
```

That's it! When the compare job will be executed, it will post a comment in the current pull-request with the difference
between the two resultset files.

For a full example of an open source project using it, take a look at: https://github.com/pupeno/repeater_world

## Cache .resultset.json

You can use the cached resultset file for comparison. To cache the resultset file that generated from the `build-base` 
job, it will save the build time.

```yaml
  test-base:
    name: Test base branch
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ## Here we do not check out the current branch, but we check out the base branch.
          ref: ${{ github.base_ref }}

      - id: base-ref-commit
        run: echo "revision=`git rev-parse HEAD`" >> $GITHUB_ENV

      - name: Simplecov resultset cache
        id: simplecov-resultset
        uses:  actions/cache@v4
        with:
          path: coverage/.resultset.json
          key: simplecov-resultset-${{ env.revision }}

      - name: Install Ruby and gems
        uses: ruby/setup-ruby@v1
        if: steps.simplecov-resultset.outputs.cache-hit != 'true'
        with:
          bundler-cache: true

      - name: Set up database schema
        if: steps.simplecov-resultset.outputs.cache-hit != 'true'
        run: bin/rails db:migrate

      - name: Precompile assets
        if: steps.simplecov-resultset.outputs.cache-hit != 'true'
        run: bundle exec rake assets:precompile

      - name: Run tests
        if: steps.simplecov-resultset.outputs.cache-hit != 'true'
        run: bundle exec rspec

      - name: Upload Coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: base-result
          path: coverage/.resultset.json
          if-no-files-found: error
          include-hidden-files: true
```

## License

MIT
