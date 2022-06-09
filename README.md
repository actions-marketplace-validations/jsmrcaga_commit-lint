# Commit Lint
 
A simple GitHub action to lint commits in Pull Requests.

---

This GitHub action allows you to write your own commit and pull request validators.

It comes with pre-defined behaviour but you can fully customize the way it works, and even the way it reports the errors back.

To start using this action you don't need anything (not even a config file!), but you can customize it if needed. The only core concept needed to begin
playing with this action are Validators.


## Usage

```yaml
name: Lint my Pull Request commits

on:
  pull_request:
    types: [opened]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: jsmrcaga/commit-lint@v0.0.1
        with:
          commit_lint_config: ./commit-lint.config.js
          github_token: ${{ secrets.GITHUB_TOKEN }}
```


## Validators

There are two types of validators, a Commit validator and a Pull request validator.
Both are essentially the same thing. They can be functions or classes, and the only requirement is that they have a `validate` method if they are objects.

For Commit validators, the argument passed to the function/method is `{ commit, pull_request }`.

For Pull request validators, the argument passed to the function/method is `{ commits, pull_request }`.
> Note that commits is here a list of commits

| Argument | Type | Definition |
|:--------:|:----:|:----------:|
| `pull_request` | Object | The [`pull_request` event](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#pull_request) itself. |
| `commits` | Object[] | The full [list of commits from GitHub's API](https://docs.github.com/en/rest/pulls/pulls#list-commits-on-a-pull-request) |
| `commit` | Object | A single commit from GitHub's API |

**Example**

```js
class MyOwnCommitValidator {
	constructor(max_length=100) {
		this.max_length = max_length;
	}

	validate({ commit: github_commit }) {
		const { message } = github_commit.commit;
		if(message.length > this.max_length) {
			throw new Error(`Commit must be less than ${this.max_length} characters`);
		}
	}
}

// Which could also be written as a function
const myOwnValidator = ({ commit: github_commit }) => {
	const { message } = github_commit.commit;
	if(message.length > 100) {
		throw new Error(`Commit must be less than ${100} characters`);
	}
};

// And for pull requests:

class MyPullRequestValidator {
	constructor(max_commits=5) {
		this.max_commits = max_commits;
	}

	validate({ commits, pull_request }) {
		if(commits.length > this.max_commits) {
			throw new Error(`Pull request cannot have more than ${this.max_commits} commits`);
		}
	}
}
```

## Config

This action merges the default configuration and the configuration you pass to it.
This means that any validators you pass will override the default behaviour, and the default validators will not run (both for commits and pull requests).

You can however import the default configuration and use it on your own.

### Usage

You can pass the path of your configuration file as a `commit_lint_config` value to your GitHub action.
This path must be relative to the working directory when the action runs, usually the root of your project.

The path is calculated as `path.join(process.cwd(), commit_lint_config)`

### Default configuration
The default configuration uses 3 different validators and the default reporter (which `exit(1)` if needed and reports errors with GitHub commands).

The default validators are
* a word length validator (with default min 3 words and max 25 words in commit message)
* a prohibited and required words validator (which prohibits `review(s)`, `lint`, `update` and `fix(es)` words, and does not require anything by default
* and a format validator, which validates a simple format: `Scope: commmit message` (with an uppercase on the first letter, and the colon).

The default configuration can be tweaked if you don't need to add your own validators. This can be achieved by passing a `default` object to your configuration and specifying some values to use. You can check them out on the full exmaple below.

### Manual configuration

In order to use a manual configuration you just need to create a `.js` file that exports an object containing your configuration values. The example below illustrates a full configuration, but since the default configuration
is merged with your own, all values are optional.

```js

module.exports = {
	// This is used to configure the default validators and reporter
	default: {
		// if the reporter should exit(1) or not if any errors have been raised
		fail_on_errors: true,

		// minimum number of words in commit. Computed using split(\s)
		min_word_length: 45,
		// maximum number of allowed words in commit
		max_word_length: 55,

		// All words or regexp that are not allowed in commit message
		// default is: [/review(s)?/i, /^\s*lint\s*$/i, /^\s*update(s)?\s*$/i, /^\s*fix(es)?\s*$/i],
		words_not_allowed: ['dog', /something[A-Z0-9]/],
		// List of required words in commit, default is []
		words_required: ['cat', /somethingelse/],

		// A regex to match the message format. Default is /[A-Z][^\:]+\:\s.+/
		// For conventional commits you can find multiple examples online
		message_format: /some_regex/
	},

	// Specific config for the Default reporter. Redundant for fail_on_errors
	reporter_config: {
		fail_on_errors: true
	},

	// List of validators for pull request, default is []
	pull_request_validators: [
		new MyPullRequestValidator()
	],

	// List of validators for commits (ever validator runs once per commit)
	// Default is explained above
	commit_validators: [
		new MyOwnCommitValidator()
	]
}
```
