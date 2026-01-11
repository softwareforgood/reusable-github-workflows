# reusable-github-workflows




## Deploy to Pantheon - `deploy-to-pantheon.yml`

BASED ON https://github.com/pantheon-systems/example-wordpress-composer

Deploys the latest code to a Pantheon instance



### How to use

#### TL;DR, I know what I'm doing

Add a job to your deploy workflow in your project like this:

```yaml
  deploy: # this name can be anything you'd like
    name: Deploy # this name can be anything you'd like
    uses: softwareforgood/reusable-github-workflows/.github/workflows/deploy-to-pantheon.yml@v2
    with:
      TERMINUS_SITE: pantheon-site-name # The name of the site as viewable by calling `terminus site:list`
      ASSETS_ARTIFACT_PATH: path/to/assets # (optional) the path to where the artifact with id "assets" should be place in the tree
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }} # An SSH private key, corresponding to a public key saved at https://dashboard.pantheon.io/personal-settings/ssh-keys
      TERMINUS_TOKEN: ${{ secrets.TERMINUS_TOKEN }} # A Pantheon machine token, as created at https://dashboard.pantheon.io/personal-settings/machine-tokens
```

#### I don't have a deploy workflow and need help.

##### The info you'll need

To get started, you'll need three pieces of info.

First, you'll need your terminus site name. Once logged in at the terminus CLI, you can get that by calling `terminus site:list`.
Once you find the site you're looking for, you need the value in the `Name`; that's your Terminus site name.

Next, need you'll need an SSH private key which is authorized for your account. You would have saved the associated SSH public key
at https://dashboard.pantheon.io/personal-settings/ssh-keys.

> I recommend using a SSH private key which is only used for deploys. That allows you to easily remove it later
> if you want to stop all deploys for that site.

Last, you'll need a Pantheon machine token. as created at https://dashboard.pantheon.io/personal-settings/machine-tokens.

> Like for the SSH private key, I recommend this be a machine token only used for deploys.

##### Creating secrets

The SSH private key and Pantheon machine token must be kept private; if someone had them, they could perform deploys on your
behalf and remotely perform any action as you to control or delete your Pantheon site. You'll need to save these as secrets available
to the repository you want to deploy. You can do so by adding them at `Actions secrets and variables` in the Settings for your repo.

Create a repository secret named SSH_PRIVATE_KEY with your SSH private key and another named TERMINUS_TOKEN with your Pantheon machine token

> Make sure you save these as a SECRET, not a VARIABLE.

> Once you save your secrets, Github won't show you them again.

#### Add your deploy workflow

Next you'll need a deploy workflow in your repo. To do that, create the `.github/workflows/deploy.yml` with the following:

```yaml
name: Deploy # this can be any name you'd like
on:
  push:
    branches:
      - main # or whatever your default branch name is
  workflow_dispatch: # this is helpful because it allows you to run the workflow from the Github UI for debugging your workflow

concurrency:
  group: ${{ github.ref }}-deploy # this guarantees you only run one deploy at a time for
jobs:
  deploy: # this can be any identifier you like
    name: Deploy # This can be anything you'd like
    uses: softwareforgood/reusable-github-workflows/.github/workflows/deploy-to-pantheon.yml@v1
    with:
      TERMINUS_SITE: terminus-site-name # put your terminus site name here
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      TERMINUS_TOKEN: ${{ secrets.TERMINUS_TOKEN }}
```


#### Build and deploy assets (optional)

Many custom theme require compiling various assets before deploy. To do so, you'll need to build your assets in job that occurs prior to 
deploy and save them as an artifact with the name of `assets`. Here's an example of doing that.

```yaml
name: Deploy 
on:
  push:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: ${{ github.ref }}-deploy
jobs:
  build:
    name: "Build theme assets"
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4
    
      # additional build steps

      - name: Upload compiled assets
        uses: actions/upload-artifact@v4
        with:
          name: assets
          # when these paths are downloaded in deploy-to-pantheon.yml, they'll be returned to the original locations in the tree
          path: path/to/some/assets
  
  deploy: 
    name: Deploy 
    needs: [build] # this is VERY important; your build needs to have completed before deploying
    uses: softwareforgood/reusable-github-workflows/.github/workflows/deploy-to-pantheon.yml@v1
    with:
      TERMINUS_SITE: terminus-site-name
      ASSETS_ARTIFACT_PATH: path/to/some/assets
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      TERMINUS_TOKEN: ${{ secrets.TERMINUS_TOKEN }}
```

> Don't worry about whether your compiled assets are Gitignore'd. Deploy to Pantheon doesn't respect .gitignore files
> when it's creating the commit to push to Pantheon. Everything in your repo's file tree, including the assets, will be
> pushed to Pantheon on deploy.
