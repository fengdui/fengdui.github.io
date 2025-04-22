Example [Hugo](https://gohugo.io) website using [GitLab Pages](https://about.gitlab.com/stages-devops-lifecycle/pages/).

Learn more about GitLab Pages at the [official documentation](https://docs.gitlab.com/ce/user/project/pages/).

## Deprecation of container registry

As of 2025-01-16 ([MR](https://gitlab.com/pages/hugo/-/merge_requests/206)), this project relies on the Docker image of <https://docker.hugomods.com>.
The last version of Hugo uploaded in the
[container registry](https://gitlab.com/pages/hugo/container_registry)
is 0.140.2. You're advised to replace `registry.gitlab.com/pages/hugo/hugo_extended:latest`
with `hugomods/hugo:exts`.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [GitLab CI/CD](#gitlab-cicd)
- [Set `baseURL` in `config.toml`](#set-baseurl-in-configtoml)
- [Building locally](#building-locally)
- [Use a custom theme](#use-a-custom-theme)
    - [Use a custom theme using a Hugo module](#use-a-custom-theme-using-a-hugo-module)
- [`hugo` vs `hugo_extended`](#hugo-vs-hugo_extended)
- [GitLab User or Group Pages](#gitlab-user-or-group-pages)
- [Did you fork this project?](#did-you-fork-this-project)
- [Troubleshooting](#troubleshooting)
    - [CSS is missing!](#css-is-missing)
    - [Minify the assets](#minify-the-assets)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## GitLab CI/CD

This project's static Pages are built by [GitLab CI/CD](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/),
following the steps defined in [`.gitlab-ci.yml`](.gitlab-ci.yml).

## Set `baseURL` in `config.toml`

Update the `baseURL` setting in your `config.toml` file from "https://pages.gitlab.io/hugo/" to your deployed Pages URL.

## Building locally

To work locally with this project, you'll have to follow the steps below:

1. Fork, clone or download this project.
1. Install [`git`](https://docs.gitlab.com/ee/topics/git/how_to_install_git/), [`go`](https://go.dev/doc/install), and [`npm`](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm).
1. [Install](https://gohugo.io/getting-started/installing/) Hugo.
1. Install the theme as a Hugo module:

   ```shell
   hugo mod get -u github.com/adityatelange/hugo-PaperMod
   ```

1. Preview your project:

   ```shell
   hugo server
   ```

1. Add content.
1. Optional. Generate the website:

   ```shell
   hugo
   ```

Read more at Hugo's [documentation](https://gohugo.io/getting-started/) and
how to [use the theme](https://github.com/adityatelange/hugo-PaperMod/wiki).

## Use a custom theme

Hugo supports a variety of themes.

Visit <https://themes.gohugo.io/> and pick the theme you want to use. In the
Pages example, we use <https://themes.gohugo.io/themes/hugo-papermod/>.

### Use a custom theme using a Hugo module

The example [`.gitlab-ci.yml`](.gitlab-ci.yml) uses Hugo modules to import the theme.

To use your own theme, the following steps will help you recreate the hugo modules
that include the information of your theme. You must perform them locally:

1. Edit `config.toml` and comment out the already existing theme:

   ```plaintext
   #theme = ["github.com/adityatelange/hugo-PaperMod"]
   ```

1. Remove `go.mod` and `go.sum`:

   ```shell
   rm -f go.mod go.sum
   ```

1. Recreate the theme module:

   ```shell
   hugo mod init gitlab.com/pages/hugo
   ```

1. Recreate the checksum:

   ```shell
   hugo mod get -u github.com/adityatelange/hugo-PaperMod
   ```

1. Edit `config.toml` and add the theme back:

   ```plaintext
   theme = ["github.com/adityatelange/hugo-PaperMod"]
   ```

1. Finally, edit `.gitlab-ci.yml`, and replace the `THEME_URL` variable with the URL of your theme:

   ```yaml
   THEME_URL: "github.com/adityatelange/hugo-PaperMod"
   ```

1. Remove the files under `content/` as they might be specific to the default theme.
1. Add content under `content/`.
1. Commit all the changes and push them to your repository.

## `hugo` vs `hugo_extended`

**NOTE:**
As of 2025-01-16, this project relies on the Docker image of <https://docker.hugomods.com>.
The last version of Hugo uploaded in the
[container registry](https://gitlab.com/pages/hugo/container_registry)
is 0.140.2. You're advised to replace `registry.gitlab.com/pages/hugo/hugo_extended:latest`
with `hugomods/hugo:exts`.

The [container registry](https://gitlab.com/pages/hugo/container_registry)
contains two kinds of Hugo Docker images, `hugo` and
`hugo_extended`. Their main difference is that `hugo_extended` comes with
Sass/SCSS support. If you don't know if your theme supports it, it's safe to
use `hugo_extended` since it's a superset of `hugo`.

The Container Registry contains three repositories:

- `registry.gitlab.com/pages/hugo`
- `registry.gitlab.com/pages/hugo/hugo`
- `registry.gitlab.com/pages/hugo/hugo_extended`

`pages/hugo:<version>` and `pages/hugo/hugo:<version>` are effectively the same.
`hugo_extended` was created afterwards, so we had to create the `pages/hugo/` namespace.

See [how the images are built and deployed](https://gitlab.com/pages/hugo/-/blob/707b8e367cdea5dbf471ff5bbec9f684ae51de79/.gitlab-ci.yml#L36-47).

## GitLab User or Group Pages

To use this project as your user/group website, you will need to perform
some additional steps:

1. [Rename your repository](https://docs.gitlab.com/ee/user/project/settings/#rename-a-repository) to `namespace.gitlab.io`, where `namespace` is
   your `username` or `groupname`.
1. Change the `baseURL` setting in your `config.toml`, from `"https://pages.gitlab.io/hugo/"` to `baseURL = "https://namespace.gitlab.io"`.
   Proceed equally if you are using a custom domain: `baseURL = "https://example.com"`.

Read more about [GitLab Pages for projects and user/group websites](https://docs.gitlab.com/ce/user/project/pages/getting_started_part_one.html).

## Did you fork this project?

If you forked this project for your own use, please go to your project's
**Settings** and remove the forking relationship, which won't be necessary
unless you want to contribute back to the upstream project.

## Troubleshooting

### CSS is missing!

That means two things:

- Either that you have wrongly set up the CSS URL in your templates.
- Or the [`baseURL`](https://gohugo.io/getting-started/configuration/#baseurl) in [`config.toml`](/config.toml) is not corretly set. For more information see issue https://gitlab.com/pages/hugo/-/issues/18.

### Minify the assets

If you want to minify the assets (JS, CSS, images, etc.), take a look at the [Hugo documentation](https://gohugo.io/getting-started/configuration/#configure-minify) and at this [merge request](https://gitlab.com/pages/hugo/-/merge_requests/79).
