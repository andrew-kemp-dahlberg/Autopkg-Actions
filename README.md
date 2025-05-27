#### About

autopromote.py is run on cron on a munki_repo server. It promotes packages between
catalogs in an order and on a schedule configured in autopromote.json.

The script works by storing a "last_promoted" datetime in the \_metadata array on all
pkginfo.plist files it promotes.

Autopromote should be run daily, and operates on a promotion schedule of days. Admins may configure a schedule of catalogs, each with a package lifetime and a force install period. Admins may also specify different promotion speeds - channels, specific patch days, and specific force install times.

### Setup

0. `mkdir .venv && virtualenv .venv && source .venv/bin/activate` (py 2.7)
1. `pip install -r requirements.txt`
2. `python3 autopromote.py` (or cron to this effect)

### Config

#### catalogs
A schedule of catalogs. Each should define the number of days a pkginfos
should live in this catalog, and the next catalog. If `next` is null, the catalog is
assumed to be the final catalog, no matter the `days` defined.

#### denylist
A dictionary of package_name: package_version_regex to refuse to promote. If the version is
`null`, promotions for all versions are blocked.

To block promoting all versions of BlueJeans and just 5.x versions
of Zoom, set:

```json
"denylist": {
	"Zoom": "5.*",
	"BlueJeans": null
}
```

#### allowlist
A dictionary of package_name: package_version_regex to refuse to promote. If a
package_name is set but, promotions which do not match package_version_regex will be blocked.

To allow only 8.x versions of Teleport, set:
```json
"allowlist": {
	"Teleport": "8.*"
}
```

#### munki_repo
 full path to the root munki_repo.

#### fields_to_copy
 when a pkginfo is promoted for the first time (no `last_promoted`
value is set), autopromote.py searchs for a previous semantic version of the pkginfo.
If found, any/all of the fields in this array are copied to the newly promoted pkginfo.

#### force_install_days
 If set, all newly promoted pkginfos receive a fresh force_install_after_date matching a T+this value. This is the default, to configure for specific packages use the `catalogs` hash.

#### patch_tuesday
An integer, 0-6, which specified the weekday to force force install dates to. For instance, if the force install date is 7 days from now, which falls on a Friday (4), and patch_tuesday is set to Tuesday (1), the force install date will be shifted by 4 days, to 11 days from now, in order to fall on the next Tuesday. This allows admins to automatically create a weekly predictability in their patch cycle.

Uses [Python's weekday implementation](https://docs.python.org/3/library/datetime.html#datetime.date.weekday) for days of the week.

| Integer Value | Day of the week |
|     :---:     | ---             |
|       0       | Monday          |
|       1       | Tuesday         |
|       2       | Wednesday       |
|       3       | Thursday        |
|       4       | Friday          |
|       5       | Saturday        |
|       6       | Sunday          |

A patch_tuesday of `null` will preclude any shift of force install dates.

#### force_install_time
 The hour and minute, T+force_install_days, at which force install will take
effect.

Format: `{"hour": int, "minute": int}`

#### enforce_force_install_time
 Have you decided 4:30 is a bad force install time? Set this value to true and configure the `force_install_time` to regulate the hour and minute, and, if set, the day (`patch_tuesday`) your package force install datetimes use.

#### enforce_force_install_date

 If `false`, `force_install_after_date` will not be updated in the output pkginfo(s). Neither a promotion period delta nor a `force_install_time` will be applied. The existing date, if set, will be preserved.

#### force_install_denylist
 A list of pkginfos (as defined in their `name` attribute) on which no force_install_after_date will ever be set.

#### channels
 Channels allow one to specify a faster or slower promotion schedule for specific packages. This is a dictionary of channel names and an int or float multiplier:

`{"channels": {"slow": 2.5}}` - this channel configuration would cause any packages in the "slow" channel to be promoted 2.5 times *slower*. This is achieved by multiplying the current promotion period by the channel's value. If your promotion period is ten days, setting the slow channel to 2.5 would increase the time between promotions to twenty-five days. For faster promotion schedules, specify a float modifier of less than 1. For example, a multiplier of `0.5` would result in a 2x faster promotion schedule.

You may add a package to a channel by adding a channel key to the pkginfo metadata dict. If no channel is specified, or if a non-float/int value is specified, the channel modifier is always 1. A package in the "slow" channel would have this in its pkginfo:

```xml
<key>_metadata</key>
<dict>
    <key>channel</key>
    <string>slow</string>
</dict>
```

If you're using [AutoPkg](https://github.com/autopkg/autopkg) you can configure this in your recipe override, so all versions of a package enter the same channel. Place the channel info in the `Input` section of your recipe override:

```xml
<key>metadata_additions</key>
<dict>
    <key>channel</key>
    <string>slow</string>
</dict>
```

This directory contains an example setup of using Github Actions to orchestrate AutoPkg and Munki.

### How it works

We've supplied an example override for Firefox.

The different workflows run on a staggered schedule to avoid merge conflicts. AutoPkg can also be run on-demand.


* `autopkg.yml` - Checks out the latest version of your autopkg overrides, installs Munki and autopkg, then clones all the upstream recipe repos. We forked Facebook's `autopkg_tools.py` script, which iterates over a list of recipes, and successful builds are pushed into a separate Git LFS repo. The build results are posted to a Slack channel so we can fix any recipe trust issues with a pull request. This also runs hjuutilainen's VirusTotalAnalyzer post-processor.

* `repoclean.yml` - Pares your Munki repo down to the two newest versions of each package.

* `autopromote.yml` - Moves your Munki packages through catalogs on a set schedule.


## GitHub Actions specifics

GitHub releases can be changed after publishing, which can make your build environment change without any indication. If an action’s repo get compromised a tag could point to malicious code. We pin the SHA1 commit hash for actions instead, since git and GitHub have robust protections against SHA1 collisions.

### Setting up your local machine

Because of how AutoPkg handles relative paths, the directory paths on your machine must match the ones on the AutoPkg runner for recipes to run properly. You can see an example of the paths and preferences we use in `autopkg.yml`

### Using this repo

1. Create an empty GitHub repo with Actions enabled.
1. Copy `workflows/` to `.github/` in this repo.
1. Create an override: `autopkg make-override recipename.munki` or if there are multiple recipes with the same filename, `autopkg make-override com.github.recipe.identifier` (be sure to place it in `overrides/`)
1. Add the recipe filename to `recipe_list.json`
1. Add the repo to `repo_list.txt`
1. Create another empty Github repo with Actions enabled. This will be your munki repo.
1. In your munki repo, create a [GitHub deploy key](https://docs.github.com/en/developers/overview/managing-deploy-keys#setup-2) with read/write access to repo.
1. Copy the name of your Munki git repo to the `Checkout your munki LFS repo` step in `autopkg.yml`
1. Add the private key for your Munki repo deploy key as a Github Actions secret named `CPE_MUNKI_LFS_DEPLOY_KEY` in your AutoPkg repo. Optionally, if using Slack, add Github Actions secrets for `SLACK_WEBHOOK_URL` (a URL to post AutoPkg results in Slack) and `AUTOPROMOTE_SLACK_TOKEN`.

### YAML support

With YAML becoming more popular for AutoPkg recipes and overrides, this runner now includes YAML support. Simply ensure the recipe list uses the full file name including a `.yaml` extension.

```json
[
  "Firefox.munki.recipe.yaml",
  "GoogleChrome.munki.recipe.yaml"
]
```


## Credits

[autopkg_tools.py](https://github.com/facebook/IT-CPE/tree/master/legacy/autopkg_tools) from Facebook under a BSD 3-clause license with modifications from [tig](https://6fx.eu) all other pieces are heavily based on [it-cpe-opensource](https://github.com/Gusto/it-cpe-opensource/tree/main) from Gust under a BSD 3-clause license.