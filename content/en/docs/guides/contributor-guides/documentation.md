---
title: Documentation
description: >
  Contributors guide for the Nephio documentation
weight: 5
---

## Framework

The Nephio documentation is built with the .md / Hugo / Docsy / Netlify framework.

* **.md:** The documentation itself is written in Markdown (,md) with some Hugo and Docsy related extensions. The .md
  files are stored and managed in a Git repository in [nephio-project/docs](https://github.com/nephio-project/docs).
* **Hugo:** [Hugo](https://gohugo.io/) is used to render the documentation fo static html pages.
* **Docsy:** [Docsy](https://www.docsy.dev/) is a theme for Hugo what we use to provide the basic structure and look
  and feel of the documentation.
* **Netlify:** [Netlify](https://www.netlify.com/) is a service to host documentation. We are hosting the Nephio    
  documentation from here.

## Creating issues

Documentation issues are handled in the nephio-project/nephio repository's [issue handler](https://github.com/nephio-project/nephio/issues).
Add the *area/docs* and the *documentation* labels to the issues created for the documentation.

## Style guide

* Use US English in the documentation
* Do not add manually a table of contents to the documents. Hugo and Docsy takes care of this.
* Do not use H1 (#) headers in the documents. Docsy generates a H1 header to every document consistent with the title
  of the document. Start the headings with H2 (##)
* Use the built in alerts for notes and alerts

  ```go-html-template
  {{%/* alert title="Warning" color="primary" */%}}
  This is a note.
  {{%/* /alert */%}}
  ```

  ```go-html-template
  {{%/* alert title="Warning" color="warning" */%}}
  This is a warning.
  {{%/* /alert */%}}
  ```
* Colors to be used when creating figures are [here](https://color.adobe.com/Nephio-secondary-colors-color-theme-0bbcdea2-0533-4ab3-812f-f752f30b5b40/)
* If you add any commands to the content inline surround the comand with backticks (\` \`), like \`ls -la\`
* Do not surround IP addresses, domain names or any other identifyers with backticks. Use italics (\* \*) to mark any
  inline IP address, domain name, file name, file location or similar.
* Whenever possible define the type of code for your code blocks
  * \```bash for all shell blocks
  * \```golang for all Go blocks
  * \```yaml for all YAML blocks
  * \``` yang for all YANG blocks
  * a full list of language identifyers is available [here](https://gohugo.io/content-management/syntax-highlighting/#list-of-chroma-highlighting-languages)
* Do not add any TBDs to the documentation. If something is missing create an [issue](https://github.com/nephio-project/nephio/issues) for it
  
## How to contribute

Follow the Nephio contribution process, based on GitHub pull requests and Prow. Contributions to the Nephio
documentation require the [Nephio CLA](https://docs.linuxfoundation.org/lfx/easycla/v2-current/contributors/corporate-contributor#github).

### Build your environment

Following the description [here](https://github.com/nephio-project/docs?tab=readme-ov-file#setting-up-the-environment).

### Tests before submitting a pull request

* Build and deploy the documentation locally
  * run `hugo serve`
* Build in a same way as Netlify is doing it
  * run `hugo --gc --minify`
* Check links
  * Install and run [linkspector](https://github.com/UmbrellaDocs/linkspector)
    ```bash
    npm install -g @umbrelladocs/linkspector
    linkinspector check -c .linkspector.yml
    ```

### Checks before submitting a pr

## FAQ

1. How do I check the documentation links on the documentation before I check in my changes?

    Install and run *linkspector*
    ```
    npm install -g @umbrelladocs/linkspector
    linkinspector check -c .linkspector.yml
    ```
