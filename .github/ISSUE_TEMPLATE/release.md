---
name: Release
about: Tracking issue for a release
title: Release 2.y.z
---
For every Scala release, make a copy of this file named after the release, and expand the variables.
Ideally this should become a script you can run on your local machine. The only missing piece is programmatic triggering of Travis-CI jobs with custom config.

Variables to be expanded in this template (or export them in a local terminal, so that you can copy/paste the commands below without replacing anything):

```bash
SCALA_VER_BASE="2.13.0"
SCALA_VER_SUFFIX="-M5"
SCALA_SHA=????????????????????????????????????????
DIST_SHA=????????????????????????????????????????
SCALA_VER="$SCALA_VER_BASE$SCALA_VER_SUFFIX"
```

Key links:
  - scala/scala milestone: https://github.com/scala/scala/milestones/2.13.?
  - scala/bug milestone: https://github.com/scala/bug/milestones/2.13.?
  - scala/scala-dev milestone: https://github.com/scala/scala-dev/milestones/2.13.?
  - Discourse topic: https://contributors.scala-lang.org/t/?
  - release notes draft: https://github.com/scala/scala-dev/blob/scala-dev/releases/2.13.?.md

### N weeks before the release
- [ ] Wind down PR queue. There has to be enough time after the last (non-trivial) PR is merged and the next phase. The core of the eco-system needs time to prepare for the final!
- [ ] Triage scala/bug and scala/scala-dev tickets
- [ ] Create next scala/scala milestone, move the magical "Merge to 2.13.x" description to it (so Scabot uses it as default for new PRs), move pending PRs
- [ ] Create next scala/bug milestone, move pending issues
- [ ] Create next scala/scala-dev milestone, move pending issues
- [ ] Check PRs assigned to the milestone, also check WIP
- [ ] Announce expected release date and current nightly "release candidate" (nightly sha-mangled version) at https://scala-ci.typesafe.com/artifactory/scala-integration/ on https://contributors.scala-lang.org/c/announcements
- [ ] Also notify Scala Center advisory board members of the upcoming release, so they can help test if they want (Seth can handle this, if asked)

### Release announcement / notes
- [ ] Notify community on https://contributors.scala-lang.org/c/announcements
- [ ] Review merged PRs, make sure release-notes label is applied appropriately
- [ ] PRs with release-notes label must have excellent title & description (title will be pasted literally in release note bullet list)
- [ ] Draft release notes (PR and self-merge, so others can comment there rather than on the commits)
    - Starting point: `gh api --paginate -X GET search/issues -f q='repo:scala/scala is:pull-request is:merged milestone:2.12.14 label:release-notes' -q '.items[] | "  * \(.title) ([#\(.number)](\(.html_url)) by [@\(.user.login)](\(.user.html_url)))"'`
- [ ] On contributors thread, link to release note file and request feedback

### N days before release
- [ ] Announce no more PRs will be merged unless last-minute regressions are found. Re-iterate current nightly sha version for testing.
- [ ] Community build
  - JDK 8: https://scala-ci.typesafe.com/job/scala-2.13.x-jdk8-integrate-community-build/????
  - JDK 11: https://scala-ci.typesafe.com/job/scala-2.13.x-jdk11-integrate-community-build/????
  - JDK 17: https://scala-ci.typesafe.com/job/scala-2.13.x-jdk16-integrate-community-build/????
- [ ] Windows build on GitHub Actions: https://github.com/scala/scala/runs/????????
- [ ] JDK 17 build on [Travis-CI (cron job)](https://app.travis-ci.com/github/scala/scala/builds): https://app.travis-ci.com/github/scala/scala/builds/????????
- [ ] Check any merged PRs accidentally assigned to the next milestone in this branch, and re-assign them to this milestone
- [ ] Merge in any older release branch
- [ ] Check module versioning (is everything in versions.properties up to date?)
  - including make sure the version of [scala-asm][] we're using is using latest [ASM][]
- [ ] On major release, bump PickleFormat version
- [ ] Close the scala/scala and scala/bug milestones

[scala-asm]: https://github.com/scala/scala-asm/
[ASM]: https://asm.ow2.io/versions.html

### Point of no return

Once sufficient time has passed since last merged PR, it's time to cut the release!

How long we wait depends on what kind of release it is. For a major release, it might be 1-2 weeks, to give core projects time to try out the preceding release candidate and/or the current candidate nightly. For point releases, assuming we've given the community ahead-of-time warning and kept them appraised of progress on the Discourse thread, and assuming the last changes merged seem sufficiently safe, we might build the next day.

Be mindful of others' schedules; even minor releases make work downstream (for Scala.js and Scala Native, for the Scala 3 team, for compiler plugin authors, and so on). It's better not to release on Friday or before a holiday.

- [ ] Make sure there are no stray [staging repos](https://oss.sonatype.org/#stagingRepositories) on sonatype
- [ ] Trigger a custom build on [travis](https://app.travis-ci.com/github/scala/scala)
  - Select the correct branch
  - Custom config: `before_script: export SCALA_VER_BASE=$SCALA_VER_BASE SCALA_VER_SUFFIX=$SCALA_VER_SUFFIX`
  - Check the build status on https://github.com/scala/scala/commits/2.13.x
  - Check that the scala/scala job also triggered a following scala/scala-dist job: https://app.travis-ci.com/github/scala/scala-dist/builds/?
  - [ ] Create the scala/scala tag locally: `git tag -s -m "Scala $SCALA_VER" v$SCALA_VER $SCALA_SHA`
  - [ ] Create scala-dist tag locally: `git tag -s -m "Scala $SCALA_VER" v$SCALA_VER $DIST_SHA`
  - [ ] Note the repos to be promoted after tag is cut (see travis log)
    - https://oss.sonatype.org/content/repositories/orgscala-lang-????
    - https://oss.sonatype.org/content/repositories/orgscala-lang-????
- [ ] Sanity check jar/pom
  - https://oss.sonatype.org/content/repositories/staging/org/scala-lang/scala-compiler/$SCALA_VER/
  - in particular, if the release was staged multiple times, double check that https://oss.sonatype.org/content/repositories/staging/ has the files from the most recent build
- [ ] Check that JARs haven't mysteriously bloated — compare sizes to previous release. We have no other backstop for this.
- Remember, tags are forever, so are maven artifacts (even staged ones, as they could end up in local caches) and S3 uploads (S3 buckets can be changed, but it can takes days to become consistent)
- [ ] Push scala/scala tag: `git push https://github.com/scala/scala.git v$SCALA_VER`
- [ ] Add release notes to https://github.com/scala/scala/releases/tag/v$SCALA_VER
- [ ] Push scala/scala-dist tag: `git push https://github.com/scala/scala-dist.git v$SCALA_VER`
- [ ] Trigger two scala-dist jobs on travis (https://app.travis-ci.com/github/scala/scala-dist) with custom config. must use full-length SHAs!
  - `before_script: export version=$SCALA_VER scala_sha=$SCALA_SHA mode=archives`: https://app.travis-ci.com/github/scala/scala-dist/builds/?
  - `before_script: export version=$SCALA_VER scala_sha=$SCALA_SHA mode=update-api`: https://app.travis-ci.com/github/scala/scala-dist/builds/?
- [ ] Promote staging repos: `st_stagingRepoPromote [scala-repo]`, `st_stagingRepoPromote [modules-repo]` (or use oss.sonatype.org web UI)

### Check availability
- [ ] Check release on sonatype: https://oss.sonatype.org/content/repositories/releases/org/scala-lang/scala-compiler/$SCALA_VER/
- [ ] Check the release on maven central: https://repo1.maven.org/maven2/org/scala-lang/scala-compiler/$SCALA_VER/

### When everything is on maven central
- [ ] Prepare PR to https://github.com/scala/scala-lang/ (using scala/make-release-notes which requires a staged release and a pushed tag)
  - `download/index.md`
  - `_config.yml` (update devscalaversion or scalaversion)
  - `index.md` (update `currentScalaVersion`)
- [ ] Prepare PR to https://github.com/scala/docs.scala-lang/
  - `api/all.md`, `_config.yml`, `_overviews/jdk-compatibility/overview.md`
- [ ] make sure the draft release notes are on GitHub tag: https://github.com/scala/scala/releases/tag/v$SCALA_VER
- [ ] Pre-announce the release on https://contributors.scala-lang.org/c/announcements
- [ ] On 2.13.0 only: (manually) update the `current` symlink for the API docs
  - https://github.com/scala/scala-dist/blob/2.13.x/scripts/jobs/release/website/update-api#L15
- [ ] Check that the API docs are published
  - http://www.scala-lang.org/api/$SCALA_VER/
  - https://docs.scala-lang.org/api/all.html ?
  - if they don't show up, possible troubleshooting steps include:
    - review the two scala-dist job logs to make sure that
      - the first one appears to have succeeded putting files in `/home/linuxsoft/archives/scala/api` on `chara.epfl.ch`
      - the second one appears to have succeeded in updating the symlink (from `2.1x.y` to $SCALA_VER)
    - ssh to chara.epfl.ch and poke around to see if things are where they should be
      - if you don't have the credential for this locally but you are able to bring jenkins-worker-publish up at `ssh jenkins-worker-publish`, then from there you can `ssh -i ~/.ssh/jenkins_lightbend_chara scalatest@chara.epfl.ch`
    - see if https://scala-webapps.epfl.ch/jenkins/view/All/job/production_scala-lang.org-scala-dist-archive-sync/ has run a job yet to sync the changes into production
      - if not, you can manually trigger a job. Seth has access to do that, probably others on the team do too. if we get stuck, Fabien can help
- [ ] Merge the scala-lang PR and the docs.scala-lang.org PR
  - [ ] wait for them to arrive on the websites and make sure they look okay
    - if the scala-lang changes don't show up, possible troubleshooting steps include:
      - see if https://scala-webapps.epfl.ch/jenkins/view/All/job/production_scala-lang.org-builder/ has run a job yet to actually publish the changes
      - see note above about permissions to trigger a job

### Modules
- [ ] build and release scala-collection-compat and other modules (or open tickets asking that the maintainers do so)
    - this work has moved to https://github.com/scala/make-release-notes/blob/2.13.x/projects-2.13.md
- [ ] if it's a 2.12.x release, publish macro paradise for the new version

### Announcements
- [ ] Scala Users discourse https://users.scala-lang.org
- [ ] add a reply on the same https://contributors.scala-lang.org thread where you announced that the release process was starting
- [ ] Tweet from [@scala_lang](https://twitter.com/scala_lang)
- [ ] https://gitter.im/scala/scala
    - [ ] if you feel like it, say something in https://gitter.im/scala/contributors too
    - [ ] Discord in addition, or instead?
- [ ] ask Seth to announce on #scala IRC

### Afterwards
- [ ] Check the release on maven search (may take a few hours):
  - https://search.maven.org/#search%7Cga%7C1%7Cg%3Aorg.scala-lang%20v%3A$SCALA_VER
- [ ] Create PR to add/update spec links on scala-lang.org (example: https://github.com/scala/scala-lang/pull/1050)
- [ ] Create a scala/scala PR to:
  - [ ] update `starr.version` in `/versions.properties`
  - [ ] update `Global / baseVersion` in `/build.sbt`
  - [ ] update `mimaReferenceVersion` in `/project/MimaFilters.scala`
  - [ ] clear out `mimaFilters` in `/project/MimaFilters.scala`, except the one(s) labeled "KEEP"
  - [ ] `spec/_config.yml`, if it's a major bump
- If it's a major bump:
  - [ ] Update `latestSpecVersion` in `spec/_config.yml` on the old branch, so that spec is marked as no longer current
  - [ ] Ditto for the nightly build and spec links in `_data/footer.yml` and `_data/doc-nav-header.yml` on docs.scala-lang.org
- [ ] Create PR to update https://github.com/lightbend/lightbend-technology-intro-doc/blob/master/docs/modules/getting-help/pages/build-dependencies.adoc
- [ ] Consider updating https://docs.scala-lang.org/overviews/jdk-compatibility/overview.html
- [ ] (Lightbend) Publish scala-fortify-plugin
- [ ] (Lightbend) Update whatever we replaced WhiteSource with?
- [ ] (Lightbend) Notify eng-updates
- [ ] Close this ticket and close the scala-dev milestone
