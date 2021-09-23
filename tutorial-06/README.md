# TUTORIAL 06: creating a Business Model Canvas and export it into PowerPoint

Tutorial on how to create a Business Model Canvas and export it into PowerPoint

## Table of Contents

- [Project setup](#project-setup)
- [Available scripts](#available-scripts)

## Project setup

This project was bootstrapped with [create-lxr](https://github.com/fazendadosoftware/leanix/tree/master/packages/create-lxr).

1. Clone the repository.
1. Run `npm install` in project directory
1. Create a `lxr.json` file in project directory (please see [Getting started](https://github.com/leanix/leanix-reporting-cli#getting-started))

## Available scripts

In the project directory, one can run:

`npm run dev`

This command will start the local development server. Please make sure you have properly configured `lxr.json` first.
It will take the specified API Token from `lxr.json` and automatically do a login to the workspace.

`npm run build`

Builds the report and outputs the build result into `dist` folder.


`npm run upload`

Uploads the report to the workspace configured in `lxr.json`.
Please see [Uploading to LeanIX workspace](https://github.com/leanix/leanix-reporting-cli#uploading-to-leanix-workspace).