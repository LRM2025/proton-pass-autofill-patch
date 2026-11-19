# Proton Pass Autofill Patch

An unofficial source patch for the Proton Pass desktop Chrome extension. It adds search to the login menu next to the form and can fill an empty password field using the email already on the page.

Based on [Proton Pass 1.44.0](https://github.com/ProtonMail/WebClients/tree/proton-pass%401.44.0), pinned to [commit `c890377`](https://github.com/ProtonMail/WebClients/commit/c890377cb5da6c2041941cb67557e3ec367638a5). The patch was authored and marked as a modification on November 19, 2026.

## What changes

- **Fill by email.** Click an empty password field or Tab to it from the email field. If one eligible login matches the full email and its saved HTTPS origin matches the page, the extension fills the password field. The email stays as entered, and you submit the form yourself.
- **Search in the form menu.** Search the site's login suggestions by email address, username, or item name. Clear the search to see all suggestions for the site. This also lets you choose manually when several logins match.
- **Fill safeguards.** Automatic fill requires an unlocked vault, autofill enabled for the site, and a recognized login form in the top-level page. The code checks that the email and target field are still current when the result comes back. It skips filled password fields and cross-origin form actions.

## Apply and build

This repository provides the source patch and instructions. Build from the official Proton source before loading it. You'll need Git and the Node.js and pnpm versions declared in [Proton's root package.json](https://github.com/ProtonMail/WebClients/blob/proton-pass%401.44.0/package.json).

```bash
git clone https://github.com/LRM2025/proton-pass-autofill-patch.git
git clone --depth 1 --branch proton-pass@1.44.0 \
  https://github.com/ProtonMail/WebClients.git proton-webclients
cd proton-webclients
test "$(git rev-parse HEAD)" = c890377cb5da6c2041941cb67557e3ec367638a5
patch=../proton-pass-autofill-patch/proton-pass-1.44.0-autofill.patch
git apply --check --index "$patch"
git apply --index "$patch"
```

Proton's public checkout at this tag references internal monorepo test dependencies. The extension can be built from a pruned workspace with dependencies verified against the included lockfile. Run this from the patched `proton-webclients` directory.

```bash
git_dir=$(git rev-parse --absolute-git-dir)
build_dir=$(mktemp -d "$git_dir/proton-chrome.XXXXXX")
turbo_version=$(node -p 'require("./package.json").devDependencies.turbo')
pnpm dlx "turbo@$turbo_version" prune proton-pass-extension --out-dir "$build_dir"

git ls-files ':(top,glob)*' | awk 'index($0, "/")==0' | while IFS= read -r file; do
  [ -e "$build_dir/$file" ] || cp "$file" "$build_dir/$file"
done
sed -E 's|:registry=.*|:registry=https://registry.npmjs.org/|' .npmrc > "$build_dir/.npmrc"

cd "$build_dir"
CI=true pnpm --filter 'proton-pass-extension...' install --frozen-lockfile --ignore-scripts
cd applications/pass-extension
BUILD_TARGET=chrome RELEASE=true pnpm run build:extension
printf 'Load this folder in Chrome: %s/dist\n' "$PWD"
```

## Load in Chrome

1. Open `chrome://extensions/` in a **separate desktop Chrome test profile**. The default build uses Proton's standard Chrome extension ID, so use a profile without the official Proton Pass extension already installed.
2. Enable **Developer mode** in the top-right corner. This is required for Chrome to show **Load unpacked**.
3. Click **Load unpacked** and select the `dist` folder printed by the build command. Keep that folder in place while the extension is installed.
4. Open **Proton Pass (Private Build)** and sign in to a test account. Try one login with a unique matching email and one search with several suggestions.

The unpacked build needs manual updates. For a new Proton release, check the patch against that release and rerun the build and tests before loading it. See [Chrome's Developer mode guide](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world#load-unpacked) and [Proton's build instructions](https://github.com/ProtonMail/WebClients/tree/proton-pass%401.44.0/applications/pass-extension).

## iPhone and iPad

This patch targets desktop Chrome. On iPhone or iPad, use the [official Proton Pass app with iOS AutoFill](https://proton.me/support/pass-setup-ios). The inline search and email-based selection in this patch are desktop Chrome features.

## Verification

The source passes TypeScript checking and 62 tests in four focused suites for candidate selection, the search UI, focus handling, and field filling. These include duplicate email matches, a locked vault, an incompatible origin, and separate email and username fields.

## License

This repository is an independent modification of [ProtonMail/WebClients](https://github.com/ProtonMail/WebClients/tree/proton-pass%401.44.0). The source changes are distributed under the [GNU General Public License, version 3](LICENSE). The original code and project names belong to their respective owners.
