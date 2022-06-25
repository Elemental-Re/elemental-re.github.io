---
title: "FoundryVTT Module Development: Packaging Project Files using Ghost’s Gulp+Rollup Preset"
layout: post
---

Lately much of my time has been devoted to my pet project [Crossblade](https://github.com/Elemental-Re/crossblade), a [Foundry VTT](https://foundryvtt.com/) module written in Typescript. The entire project has been quite an undertaking for me as an amateur developer, and while I’m not COMPLETELY lost with most of the concepts involved, every step of the process has been a learning experience to some degree.


The project itself was bootstrapped with the excellent [Foundry Factory](https://github.com/ghost-fvtt/foundry-factory) tool by [Ghost](https://github.com/ghost-fvtt) using the [Gulp + Rollup Preset](https://github.com/ghost-fvtt/foundry-factory/blob/main/src/presets/ghost-gulp-rollup). This was very valuable for me starting out, because I really had no idea where to begin initially. However, having someone else set up the project for me did result in a significant blind spot in my understanding of the project build process. I knew I could build the project using commands like `npm run build` or `npm run build:watch` from the command line, but how those commands actually built the project was basically a black box as far as my understanding went.

### The Goal

I recently found myself wanting to add a changelog to my project. Creating the file, `CHANGELOG.md` was simple enough, but I wanted to get as much use out of it as possible.

For example, the [Module Management+](https://foundryvtt.com/packages/module-credits) module is apparently able to detect when the changelog updates and display a popup for GMs when a world is loaded, showing any modules that have new changes since the last time the world was loaded.

![Module Management+](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/module-management+.png "Module Management+")

### Getting to Work

Initially I wasn’t able to get my module’s changelog to be detected by Module Management+. My initial assumption was that it was pulling the changelog from `module.json` so I added the following line pointing to the file on my project’s repository.

```json
"changelog": "https://github.com/Elemental-Re/crossblade/blob/main/CHANGELOG.md" 
```

This didn’t do any good. I had to dig around for an embarrassingly long time and look at other modules that Module Management+ displayed changelogs for to discover that it actually looks for a copy of the file `CHANGELOG.md` in the module’s release files. It also looks for a `README.md` file, and if it detects either it also adds a button to view them in the Module Management dialog.

![Readme Button](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/readme-button.png "Readme Button")

So I wanted to figure out how to get these files included into the release. In the very specific case of my project using the [Gulp + Rollup Preset](https://github.com/ghost-fvtt/foundry-factory/blob/main/src/presets/ghost-gulp-rollup), “the release” is basically the files that get copied to the `dist` folder every time the project is built. I figured it had something to do with gulp as every time the project was built using `npm build` or `npm build:watch` the following line was printed to the console:

```
Using gulpfile ~\Projects\Foundry VTT\modules\crossblade\gulpfile.cjs
```

There was still some concern as to whether or not this was all I needed to do, as I wasn’t entirely sure how the release files were generated when creating a new tag in the GitHup repo, but I was pretty sure this was at least one step of that process as well, so I took a peek at the file.

### Gulp and npm

Looking at the file `gulpfile.cjs` I saw that it was broadly broken up into four different sections, `CONFIGURATION`, `BUILD`, `CLEAN`, and `LINK`. Of these, `BUILD` was by far the most extensive. `CONFIGURATION` mostly declared a number of variables while the other three contained some methods that corresponded to steps of the build process I always see output to the console, like `buildCode` or `clean`. Some of these were run as part of another method, whereas others were apparently meant to be run on their own.

I was curious as to how these methods were being called as part of the build process. The short version is that the bottom of `gulpfile.cjs` contained the following statement:

```jsx
module.exports = {
	watch,
	build,
	clean,
	link,
};
```

These exports were then referenced in `package.json`, which was what defines the project as far as npm is concerned. I saw that the arguments I had been passing to `npm run` all this time were defined by:

```jsx
"scripts": {
	"build": "gulp build",
	"build:watch": "gulp watch",
	"link-project": "gulp link",
	"clean": "gulp clean",
	"clean:link": "gulp link --clean",
	...
},
```

I still had some questions but I had at least gotten my bearings. It was enough to start modifying the build process.

## Editing the gulpfile

### copyFiles

It was pretty easy to figure out that the `copyFiles` method of the `BUILD` section was what I needed to modify first. I wasn’t building code or stylesheets, I just wanted to copy a couple of files over as is. The method looked like this:

```jsx
async function copyFiles() {
  for (const file of staticFiles) {
    if (fs.existsSync(`${sourceDirectory}/${file}`)) {
      await fs.copy(`${sourceDirectory}/${file}`, `${distDirectory}/${file}`);
    }
  }
}
```

The method referenced three variables, which were defined at the top of the gulpfile in the `CONFIGURATION` section.

```jsx
const sourceDirectory = './src';
const distDirectory = './dist';
const staticFiles = ['assets', 'fonts', 'lang', 'packs', 'templates', 'module.json'];
```

I couldn’t quite just add the additional files I wanted to copy to the `staticFiles` variable, because all of those were assumed to be under the `./src` directory. I needed to add a couple more variables and my own loop. After a couple of attempts I managed to put together the following.

```jsx
const projectDirectory = '.';
const projectFiles = ['README.md', 'CHANGELOG.md'];
```

```jsx
async function copyFiles() {
  for (const staticFile of staticFiles) {
    if (fs.existsSync(`${sourceDirectory}/${staticFile}`)) {
      await fs.copy(`${sourceDirectory}/${staticFile}`, `${distDirectory}/${staticFile}`);
    }
  }
  for (const projectFile of projectFiles) {
    if (fs.existsSync(`${projectDirectory}/${projectFile}`)) {
      await fs.copy(`${projectDirectory}/${projectFile}`, `${distDirectory}/${projectFile}`);
    }
  }
}
```

This seemed to work: both `CHANGELOG.md` and `README.md` were now being copied to the `dist` folder when executing `npm run build`.

### clean

I also noticed that the `clean` step of the build process wasn’t cleaning up these new files. I had to edit that as well. Here’s what it looked like initially:

```jsx
async function clean() {
  const files = [...staticFiles, 'module'];

  if (fs.existsSync(`${stylesDirectory}/${name}.${stylesExtension}`)) {
    files.push('styles');
  }

  console.log(' ', 'Files to clean:');
  console.log('   ', files.join('\n    '));

  for (const filePath of files) {
    await fs.remove(`${distDirectory}/${filePath}`);
  }
}
```

The change here was much simpler. Everything to be cleaned up was together in the `distDirectory` so all I needed to do was add the values of the new `projectFiles` variable to the `files` constant so they could be iterated over as well..

```jsx
const files = [...staticFiles, ...projectFiles, 'module'];
```

### watch

The last thing I noticed was that changes to `README.md` and `CHANGELOG.md` were not triggering a new build when running `npm build:watch`. Arguably, there was no real need to fix this, it’s not like I expected to edit them regularly during the build process, but I had come this far, so I figured I might well go all the way. In order to resolve this I had to register the files under the `watch` method. This is what it looked like initially:

```jsx
function watch() {
  gulp.watch(`${sourceDirectory}/**/*.${sourceFileExtension}`, { ignoreInitial: false }, buildCode);
  gulp.watch(`${stylesDirectory}/**/*.${stylesExtension}`, { ignoreInitial: false }, buildStyles);
  gulp.watch(
    staticFiles.map((file) => `${sourceDirectory}/${file}`),
    { ignoreInitial: false },
    copyFiles,
  );
}
```

I tweaked the last `gulp.watch` statement, pulling out the mapping of static files to paths to a constant and adding another for my new project files so that I could `concat` them together.

```jsx
const staticFilesPaths = staticFiles.map((staticFile) => `${sourceDirectory}/${staticFile}`);
const projectFilesPaths = projectFiles.map((projectFile) => `${projectDirectory}/${projectFile}`);
gulp.watch(staticFilesPaths.concat(projectFilesPaths), { ignoreInitial: false }, copyFiles);
```

This seemed to do the trick. Changes to the two files were triggering the `copyFiles` method and the changed files would be immediately copied to the `dist` folder. At this point it was just a matter of whether or not this would be picked up by the build process in github.

## Github Actions

I thought it best to just take a look at how previous builds had been run to get an idea of the process rather than just push my code and hope for the best. I knew I could check this in the actions tab of the repo, but I hadn’t looked at it in-depth before.

The first thing I noticed was that there was a list of each kind of workflow to the left of the list of all previous workflow runs.

![Workflow List](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/workflow-list.png "Workflow List")

‘Release’ was clearly what I was looking for. Clicking on that, I immediately noticed a filename, `release.yml` listed underneath the heading of the tab.

![Release Workflow](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/release-workflow.png "Release Workflow")

This was probably the file that defined the workflow. I couldn’t see the contents from the actions tab, but I took a peek at the project files and found it under `/.github/workflows/release.yml`. Inside I found `npm run build` as part of the following section of data:

```yaml
- jobs:
	- build:
		- steps:
			- name: Build
			  run: npm run build
```

This was exactly what I ran to build the project and should be calling the gulp tasks that I modified, so it was reasonable to assume that my changes should be picked up as part of GitHub’s build process.

## The Results

I pushed the changes and created a new tag, then installed the new version to my copy of Foundry to test it out. Upon loading into my world I was immediately greeted by a popup from Module Management+ showing my changelog.

![Crossblade Changelog](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/crossblade-changelog.png "Crossblade Changelog")

I also noticed it was showing a button for the module's Readme in the actual module management dialog.

![Crossblade Readme](/assets/images/2022-06-25-packaging-project-files-using-ghost’s-gulp+rollup-preset/crossblade-readme.png "Crossblade Readme")

Everything seemed to be working exactly I had hoped. It was quite a process for a change that didn’t modify any of the modules core functions, but for those with Module Management+ installed, hopefully this will help them to keep updated to changes to Crossblade, and I learned a lot that I can easily apply to other projects in the future.
