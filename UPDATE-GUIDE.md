# Update Guide

This is a personal fork of [zhukunpenglinyutong/idea-claude-code-gui](https://github.com/zhukunpenglinyutong/idea-claude-code-gui). This guide explains how to pull in upstream changes safely — including the security review step that's important given the trust concerns documented in [SECURITY-REVIEW.md](SECURITY-REVIEW.md).

## Local environment

- Upstream clone: `C:\Data\Repository\External\idea-claude-code-gui` (origin → upstream repo)
- Our fork: `C:\Data\Repository\PrivJimmie\idea-claude-code-gui` (origin → privjimmie/idea-claude-code-gui)
- The fork has the upstream registered as `upstream` remote

## Our customizations (preserve these across updates)

1. **`src/main/resources/META-INF/plugin.xml`** — plugin name has ` (Forked by Jimmie)` suffix
2. **`README.md`** — "Fork by Jimmie" section prepended at top
3. **`SECURITY-REVIEW.md`** — security audit notes
4. **`UPDATE-GUIDE.md`** — this file
5. **`.gitignore`** — tweaked to allow committing `build/distributions/*.zip`
6. **`build.gradle`** — Java toolchain set to **21** (upstream uses 17). Required because Rider 2025.3.2 ships with class file version 65+, and a JDK 17 toolchain can't compile against it. Search for `JavaLanguageVersion.of(21)` — there are two occurrences.
7. **`build/distributions/idea-claude-code-gui-*.zip`** — committed prebuilt binary so users don't need to build

## Step-by-step update process

### 1. Sync the upstream clone (optional but recommended)

```bash
cd /c/Data/Repository/External/idea-claude-code-gui
git pull origin main
```

### 2. Fetch upstream into our fork

```bash
cd /c/Data/Repository/PrivJimmie/idea-claude-code-gui
git fetch upstream
```

If the `upstream` remote isn't set up:
```bash
git remote add upstream https://github.com/zhukunpenglinyutong/idea-claude-code-gui.git
git fetch upstream
```

### 3. Review what's coming

```bash
git log --oneline HEAD..upstream/main
git diff --stat HEAD..upstream/main
```

Get a sense of: number of commits, files touched, any package.json/build.gradle/dependency changes.

### 4. Merge upstream

```bash
git merge upstream/main --no-edit
```

If there are merge conflicts:
- The most likely conflicts will be in `build.gradle` (toolchain `21` vs upstream's `17`) and `README.md`
- Always **keep our `21`** for the toolchain — JDK 17 won't compile against modern Rider classes
- For README, keep our "Fork by Jimmie" header at the top, then append upstream's changes below

### 5. Verify customizations survived

```bash
grep "Forked by Jimmie" src/main/resources/META-INF/plugin.xml
head -5 README.md
grep "languageVersion" build.gradle    # Both should show 21
```

### 6. Run a delta security review

This is **critical**. Don't skip this. Even small upstream changes can introduce risks.

Spawn parallel review agents (use the Agent tool with `subagent_type=Explore`) to check:

1. **New dependencies** — diff `build.gradle`, `webview/package.json`, `ai-bridge/package.json`. For any new package, verify it's open source, check weekly download counts on npmjs.com, and confirm no Chinese mirrors or private registries were added.
2. **New network calls** — search for hardcoded URLs (excluding the known-safe list: `api.anthropic.com`, `api.openai.com`, `127.0.0.1`, `localhost`, `services.gradle.org`, `registry.npmjs.org`, `github.com`, `jetbrains.com`). Watch for any `.cn` domains, alibaba/tencent/baidu endpoints, or unfamiliar proxy targets.
3. **New credential-handling code** — look for changes touching API keys, tokens, environment variables, or `~/.claude/settings.json` access patterns.
4. **Suspicious patterns** — `eval()`, `Base64.decode`, dynamic code loading, new native library bindings, new shell execution paths.

Quick CLI commands for the delta check:
```bash
git diff <previous-tag>..HEAD -- '*.gradle' 'package.json' '*/package.json'
git log <previous-tag>..HEAD --oneline -- 'src/main/java/com/github/claudecodegui/dependency/SdkDefinition.java'
```

Update `SECURITY-REVIEW.md` with a new dated section summarizing what was reviewed and the findings (even if "no new risks"). Keep the historical entries — they show the audit trail.

### 7. Update build dependencies and rebuild

```bash
cd webview && npm install && cd ..
cd ai-bridge && npm install && cd ..

export JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot"
export PATH="$JAVA_HOME/bin:$PATH"
./gradlew buildPlugin -PtargetIde=RD
```

Build output: `build/distributions/idea-claude-code-gui-<version>.zip`

### 8. Update README + remove old binary

- Update version references in `README.md` (the two `idea-claude-code-gui-X.X.X.zip` mentions)
- Remove the old version's zip from git: `git rm build/distributions/idea-claude-code-gui-<old-version>.zip`

### 9. Test the build in Rider

1. **Rider** > **Settings** > **Plugins**
2. Uninstall the old version of the plugin if installed
3. Gear icon > **Install Plugin from Disk...** > select the new zip
4. Restart Rider
5. Open the tool window, send a quick test message, verify it works

### 10. Commit and push

```bash
git add -A
git commit -m "Update to upstream v<X.X.X> with security review"
git push
```

Include the new zip, updated SECURITY-REVIEW.md, README.md, and any merge commits.

## Common gotchas

- **JDK version drift** — Rider's bundled JBR auto-updates and may go beyond what Gradle 8.14 supports. If the build complains about "class file major version XX", install a standalone JDK 21 from Temurin instead of relying on JBR.
- **gitignore rules** — the `build/` directory is gitignored, so `build/distributions/*.zip` requires the explicit unignore rules in our `.gitignore`. Don't simplify those rules without testing that `git status` still sees the zip.
- **Plugin name suffix** — keep ` (Forked by Jimmie)` in `plugin.xml` so the installed plugin is visually distinguishable from the marketplace version.
- **Conflict in CHANGELOG.md / version bumps** — usually safe to accept upstream's version. Don't try to keep our own version numbering.
