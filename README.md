# teamshares-monorepo-buildpack

Custom buildback for deploying apps within the [teamshares monorepo](https://github.com/teamshares/monorepo) on heroku.

### History
- We were previously using a brittle flow that only allowed using a single buildpack (which meant we couldn't e.g. specify specific node versions).
- Downstream buildpacks (`heroku/nodejs` + `heroku/ruby`) assume the app lives in the root directory (i.e. searches for `Gemfile` in root, heroku has no support for nested apps)
- All the existing monorepo buildpacks I found are some variation of "take the specified subdirectory, move it to root, remove everything else".
- This didn't work for us because we explicitly depend on sibling folders (that those buildbacks would remove) from our yarn workspaces


  Options:
  - Remove sibling dependencies by using private (e.g. gemfury) repositories
  - The file path dependencies *work*, just write a custom buildpack to move them under the directory we're keeping.

  Solution:
  - We chose option 2 (at least for now); that's the origin story for the buildpack you're looking at now.

### What does it do now?
- This [teamshares-specific buildpack](https://github.com/teamshares/teamshares-monorepo-buildpack) kicks off the flow by moving `teamshares_components` from root to under the directory we're keeping, then rewriting some configs so the paths stay current. Now multiple buildpacks are supported, so:
  - We can use ~~any of the standard promote-this-subdir-of-monorepo-to-root buildpacks~~ (it was brittle and "subject to update at any time", so we eventually inlined this functionality directly into this buildpack), and then
  - `heroku/nodejs` allows us to explicitly control node version via `package.json` (this buildpack exports updated node paths for any subsequent paths)
  - `heroku/ruby` handles installing rubygems

### Downsides?
One currently-known downside: we're switching from a yarn workspace context in development (multiple separate apps' dependencies combined into single root-level `yarn.lock`) to deploying just a single subdir in production. Naive approach is to copy that `yarn.lock` into the subdir, but that makes the build phase fail -- `heroku/nodejs` runs yarn with `--frozen-lockfile`, but the lockfile _should_/_will_ change now that we're using a different `package.json` and no longer need all the dependencies for the whole workspace.

Hopefully eventually there'll be some sort of `yarn workspace eject` or similar command we can use -- there are some very long github discussions ([e.g.](https://github.com/yarnpkg/berry/issues/1223)) without a clear resolution.

For now I've worked around this by adding a `heroku-prebuild` script to run `yarn install` in the subdirectory. This now *works*, but note it means **we are no longer guaranteed that the versions specified in local `yarn.lock` are the same that will be installed in production** (although all our JS dependencies are currently specified with either patch-level (`~`) or minor version (`^`) [limiters](https://docs.npmjs.com/about-semantic-versioning), so I think this is a sane tradeoff).
