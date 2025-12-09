<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.1/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.3/README.md">Next ➡️</a></td>
  </tr>
</table>

# 2. Creating the website
To create the website published during this laboratory we will leverage [Hugo](https://gohugo.io/).
Hugo is an open-source static site generator, which simplifies the creation of static websites, such as personal pages, blogs, documentation resources, company or product pages.
It features a wide variety of predefined **templates** (a.k.a. themes), while
**content can be expressed in markdown**, which is then automatically
converted by Hugo into the actual *html* pages.

## 2.1 Scaffolding the website structure

In order to create a new Hugo website, it is possible to leverage the `hugo new site` command, which initializes a new project and scaffolds the appropriate directory structure:
```sh
hugo new site path/to/website
```
Looking at the content of the directory, the `config.toml` file states
various configuration directives (e.g., the base URL where the website
will be published, the theme to use, …). The `content` directory
contains the markdown files about the different website sections,
`static` stores all static content, including images, CSS, javascript,
…, and `themes` embeds the files about the selected theme (once
downloaded).

To prevent issues when building the final website (e.g., missing CSS files), let explicitly configure the website base URL to be relative:
```sh
sed -i 's|baseURL = "http://example.org/"|baseURL = "/"|' path/to/website/config.toml
```
> [!NOTE]
> It is suggested to initialize the resulting directory as a Git repository, in order to keep track of the different modifications and
simplify the download of the selected theme.
> The Git repository can be initialized as follows (to be issued from `/path/to/website`):
> ```sh
> git init
> git add .
> git commit -m "Initial commit"
> ```

## 2.2 Configuring a theme

With the initial structure in place, it is now time to select a theme
for our website. Available themes can be explored on the [official page](https://themes.gohugo.io/): you can freely choose the one you prefer.
The following description assumes you selected the *Learn* theme, but the steps are definitely similar regardless of the choice (additional information is provided by each theme’s subpage).

Once selected, it is necessary add the theme to the website directory
structure. While it is possible to download it from the repository as a
zip file and extract the content in the `themes` directory, we suggest
to leverage **git submodules**. Git submodules allow to store a git
repository (e.g., the one of the theme), as a subdirectory of another
git repository (e.g., our website), though maintaining separate version
control information. Overall, this allows to easily update the theme
files whenever a new version is available upstream, while decoupling
them from the main website repository.

A new submodule (for the *Learn* theme) can be initialized as follows
(the content of the theme is downloaded in the `themes/hugo-learn`
folder):
```sh
git submodule add https://github.com/matcornic/hugo-theme-learn.git themes/hugo-learn
```

Finally, it is necessary to configure the theme in the `config.toml`
file, adding the appropriate line:
```toml
theme = "hugo-learn"
```
> [!TIP] 
> This might be a good moment to commit your changes.

## 2.3 Adding the site content

Now, it is possible to proceed adding the actual content to your
website. Each section of the website is represented by a different
folder inside the `content` directory, and it shall contain one or
multiple markdown files corresponding to the different section pages.
One page shall always be named `_index.md`, and it represents the root
page of the given section. 
> [!WARNING]
> The final appearance of the website depends on the theme selected during the previous step.
> Different themes might also rely on different content structures: refer to their respective documentation for additional information.

To create the content files, you can leverage once more the appropriate
`hugo` command, which takes care of configuring the necessary metadata
information at the beginning of each one (based on pre-defined
templates). Specifically, to create two sections (e.g., `basic` and
`advanced`), each composed of two pages (e.g., `first` and `second`), it
is possible to proceed as follows:
```sh
# Modify the template to avoid marking the pages as draft
sed -i 's/draft: true/draft: false/' archetypes/default.md

# Create the root "index" file
hugo new _index.md

# Create the "basic" section
hugo new basic/_index.md
hugo new basic/first.md
hugo new basic/second.md

# Create the "advanced" section
hugo new advanced/_index.md
hugo new advanced/first.md
hugo new advanced/second.md
```

With the website structure in place, you can add the actual content,
editing the different files. As mentioned earlier, each file starts with
a metadata section, which includes the title and possible additional
information (e.g., a priority, to enforce a different sections order).
The content can be written below the metadata part, leveraging the
markdown syntax to express formatting options.

## 2.4 Building the website

While writing the different contents, it is often useful to preview the
resulting website. Hugo offers the possibility to start a local
webserver by means of the `hugo server` command (to be issued from the
root directory of the website): the site can then be accessed at
`http://localhost:1313`, and every change in the content is
automatically reflected to the preview.

Differently, to generate the files (i.e., *html* pages, *CSS*
stylesheets and *JavaScript* scripts) to be published, you can leverage
the `hugo -v` command (once more from the root directory of the
website). The outcome of the process is made available in the `public`
subdirectory.