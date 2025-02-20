---
share: true
---


/* eslint-disable no-case-declarations */

  

import path from "path";

  

import { select, input } from "@inquirer/prompts";

import chalk from "chalk";

import { Plop, run } from "plop";

  

import { executeCommand } from "./lib/command.js";

import {

  appList,

  databases,

  getApp,

  getDefaultLogOptions,

  packages,

} from "./lib/config.js";

import { getTemplateFolders } from "./lib/utils.js";

import { buildProcess } from "./tasks/build-process/build-test-deploy.js";

import { copyStandaloneFiles } from "./tasks/build-process/post-build-standalone.js";

import { buildApp } from "./tasks/build-process/run-build.js";

import { runLocalProdBuild } from "./tasks/build-process/run-local-prod-build.js";

import { createDBImage } from "./tasks/dbs/create-db.js";

import { exportCollection } from "./tasks/dbs/export-collection.js";

import { migrateDB } from "./tasks/dbs/migrate-db.js";

import { startDB } from "./tasks/dbs/start-db.js";

import { deployToCoolify } from "./tasks/deploy/deploy-coolify.js";

import { deployToVercel } from "./tasks/deploy/deploy-vercel.js";

import { generateBlocks } from "./tasks/dev/blocks/ai-blocks.js";

import { getBlockTypes } from "./tasks/dev/blocks.js";

import { checkTypes } from "./tasks/dev/check-types.js";

import { cleanMonorepo } from "./tasks/dev/clean.js";

import { connectToDocker } from "./tasks/dev/connect-docker.js";

import { createProject } from "./tasks/dev/create-project.js";

import { generateIndexFile } from "./tasks/dev/generate-index.js";

import { lintApp } from "./tasks/dev/lint.js";

import { mergeTsFiles } from "./tasks/dev/merge-files.js";

import overwriteBlocks from "./tasks/dev/overwrite-blocks.js";

import { renameToKebabCase } from "./tasks/dev/rename-to-kebab-case.js";

import { startDevMode } from "./tasks/dev/run-dev.js";

import {

  checkNextVersion,

  checkPayloadVersion,

} from "./tasks/misc/check-versions.js";

import downloadRelumeComponents from "./tasks/misc/relume.js";

import { uploadToR2, getConfig } from "./tasks/misc/upload-r2.js";

import { startTestServer } from "./tasks/testing/start-test-and-run-e2e.js";

  

/**

 * Run the interactive CLI mode

 */

export async function runInteractiveMode() {

  // Add cleanup function

  const cleanup = () => {

    console.log(chalk.green("\n\n👋 Goodbye!"));

    // Force exit after 100ms to ensure console message is shown

    setTimeout(() => process.exit(0), 100);

  };

  

  // Handle both SIGINT (Ctrl+C) and SIGTERM

  process.on("SIGINT", cleanup);

  process.on("SIGTERM", cleanup);

  

  try {

    // Fix the spread operator issues by awaiting the promises

    const blockTypes = await getBlockTypes();

  

    while (true) {

      try {

        const selectedScript = await select({

          message: "Select a script to run",

          choices: [

            { name: "<- Select ->", value: "select" },

            { name: "Development tasks", value: "dev" },

            { name: "Database", value: "db" },

            { name: "Build", value: "build" },

            { name: "Build local prod", value: "build-local-prod" },

            { name: "Connect docker", value: "connect-docker" },

            { name: "Deploy", value: "deploy" },

            { name: "Test", value: "test" },

            { name: "Build, Test and Deploy", value: "build-test-deploy" },

            { name: "Misc", value: "misc" },

            { name: "<- Exit ->", value: "exit" },

          ],

        });

  

        switch (selectedScript) {

          case "select":

            break;

  

          case "dev":

            const devChoice = await select({

              message: "Select a development option:",

              choices: [

                { name: "<- Go Back", value: "back" },

                { name: "Start Dev Environment", value: "start" },

                { name: "Lint App", value: "lint" },

                { name: "Check Types", value: "check-types" },

                {

                  name: "Generate Payload Types",

                  value: "generate-payload-types",

                },

                { name: "---", value: "---" },

                { name: "Create Project", value: "create-project" },

                { name: "---", value: "---" },

                { name: "Overwrite Blocks", value: "overwrite-blocks" },

                { name: "Generate Blocks", value: "generate-blocks" },

                { name: "---", value: "---" },

                { name: "Rename to Kebab Case", value: "rename-to-kebab" },

                { name: "Merge TS files", value: "merge-ts-files" },

                { name: "Generate Index file", value: "generate-index-file" },

                { name: "---", value: "---" },

                { name: "Clean Monorepo", value: "clean" },

              ],

            });

            if (devChoice === "back") break;

  

            switch (devChoice) {

              case "start":

                const appChoice = await select({

                  message: "Select an app to run:",

                  choices: [

                    { name: "<- Go Back", value: "back" },

                    ...appList.map((app) => ({

                      name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (appChoice === "back") break;

  

                const useCustomPort = await select({

                  message: "Use custom port?",

                  choices: [

                    { name: "No", value: false },

                    { name: "Yes", value: true },

                  ],

                });

  

                let port: string | undefined;

                if (useCustomPort) {

                  port = await input({

                    message: "Enter port number:",

                    default: getApp(appChoice)?.devPort?.toString() || "3000",

                  });

                }

  

                await startDevMode(appChoice, true, getDefaultLogOptions());

                break;

  

              case "create-project":

                const projectName = await input({

                  message: "Enter project name (or type back to go back):",

                  validate: (value) =>

                    value.length > 0 || "Project name is required",

                });

                if (projectName === "back") break;

                const templateName = await input({

                  message: "Enter template name:",

                  validate: (value) =>

                    value.length > 0 || "Template name is required",

                });

                if (projectName === "back") break;

                await createProject(projectName, templateName);

                break;

  

              case "create-collection":

                console.log(

                  "Creating collection...",

                  path.join(

                    process.cwd(),

                    "/packages/webforma-cli/plopfile.js",

                  ),

                );

                await new Promise(() => {

                  Plop.prepare(

                    {

                      cwd: process.cwd(),

                      configPath: path.join(

                        process.cwd(),

                        "/packages/webforma-cli/plopfile.ts",

                      ),

                      preload: [],

                      completion: "false",

                    },

  

                    //@ts-ignore

                    // eslint-disable-next-line @typescript-eslint/no-misused-promises

                    (env) => Plop.execute(env, run),

                  );

                });

                console.log(chalk.green("✨ Collection created successfully!"));

                break;

              case "merge-ts-files":

                const targetDirectoryForMerge = await input({

                  message: "Enter target directory. Type back to go back:",

                });

                if (targetDirectoryForMerge === "back") break;

                await mergeTsFiles(targetDirectoryForMerge);

                break;

  

                break;

  

              case "generate-index-file":

                const targetDirectoryForIndex = await input({

                  message: "Enter target directory. Type back to go back:",

                });

                if (targetDirectoryForIndex === "back") break;

                generateIndexFile(targetDirectoryForIndex);

                break;

  

              case "clean":

                await cleanMonorepo();

                break;

  

              case "check-types":

                const typeCheckApp = await select({

                  message: "Select an app to type check:",

                  choices: [

                    { name: "Go back", value: "back" },

                    { name: "All", value: "all" }, // Add this line for the "All" option

                    ...[...appList, ...packages].map((app) => ({

                      name: `${app.name} - ${"isTemplate" in app ? (app.isTemplate ? "Template" : "App") : "Package"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (typeCheckApp === "back") break;

  

                await checkTypes(typeCheckApp);

                break;

  

              case "generate-payload-types":

                const generatePayloadTypesApp = await select({

                  message: "Select an app to generate payload types:",

                  choices: [

                    { name: "Go back", value: "back" },

                    { name: "All", value: "all" }, // Add this line for the "All" option

                    ...[...appList, ...packages].map((app) => ({

                      name: `${app.name} - ${"isTemplate" in app ? (app.isTemplate ? "Template" : "App") : "Package"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (generatePayloadTypesApp === "back") break;

                //await generatePayloadTypes();

                break;

  

              case "lint":

                const appChoice3 = await select({

                  message: "Select an app to lint:",

                  choices: [

                    { name: "Go back", value: "back" },

                    { name: "All", value: "all" }, // Add this line for the "All" option

                    ...[...appList, ...packages].map((app) => ({

                      name: `${app.name} - ${"isTemplate" in app ? (app.isTemplate ? "Template" : "App") : "Package"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (appChoice3 === "back") break;

                const shouldFix = await select({

                  message: "Auto-fix linting issues?",

                  choices: [

                    { name: "No", value: false },

                    { name: "Yes", value: true },

                  ],

                });

  

                await lintApp(appChoice3, shouldFix, getDefaultLogOptions());

                break;

  

              case "generate-blocks":

                const blockChoice2 = await select({

                  message: "Select a block type:",

                  choices: [{ name: "Go back", value: "back" }, ...blockTypes],

                });

                if (blockChoice2 === "back") break;

                await generateBlocks();

                break;

  

              case "overwrite-blocks":

                const appChoice1 = await select({

                  message: "Select an app:",

                  choices: [

                    { name: "Go back", value: "back" },

                    ...appList.map((app) => ({

                      name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (appChoice1 === "back") break;

  

                const app = getApp(appChoice1);

                if (!app) throw new Error("App not found");

  

                // Fix template folders spread

                const templateFolders = await getTemplateFolders(app.name);

                const templateChoice = await select({

                  message: "Select a template:",

                  choices: [

                    { name: "Go back", value: "back" },

                    ...templateFolders,

                  ],

                });

  

                if (templateChoice === "back") break;

  

                // Fix block types spread again

                const blockChoice = await select({

                  message: "Select a block:",

                  choices: [{ name: "Go back", value: "back" }, ...blockTypes],

                });

  

                if (blockChoice === "back") break; // Fix the condition check

  

                await overwriteBlocks({

                  appName: app.name,

                  templateName: templateChoice,

                  blockName: blockChoice,

                });

                break;

  

              case "rename-to-kebab":

                const targetDirectory = await input({

                  message:

                    "Enter target directory (or press enter for current directory). Type back to go back:",

                  default: ".",

                });

                if (targetDirectory === "back") break;

                renameToKebabCase(targetDirectory);

                break;

            }

            break;

  

          case "db":

            const dbChoice = await select({

              message: "Select a database task:",

              choices: [

                { name: "<- Go Back", value: "back" },

                { name: "Start DB container", value: "start-db-container" },

                { name: "Export collection", value: "export-collection" },

                { name: "Import collection", value: "import-collection" },

                { name: "Migrate DB", value: "migrate-db" },

                {

                  name: "Create custom DB image (not used currently)",

                  value: "create-db-image",

                },

              ],

            });

            if (dbChoice === "back") break;

            switch (dbChoice) {

              case "start-db-container":

                const dbSelect = await select({

                  message: "Select a database to start:",

                  choices: [

                    { name: "<- Go Back", value: "back" },

                    ...databases.map((db) => ({

                      name: `${db?.app} - ${db?.env.toUpperCase()} - (${db?.URI})`,

                      value: db?.name,

                    })),

                  ],

                });

                if (dbSelect === "back") break;

                if (!dbSelect) throw new Error("Database name is required");

                await startDB(dbSelect);

                break;

              case "export-collection":

                const dbSelect2 = await select({

                  message: "Select a database to export collection from:",

                  choices: [

                    { name: "<- Go Back", value: "back" },

                    ...databases.map((db) => ({

                      name: `${db?.app} - (${db?.URI})`,

                      value: db?.name,

                    })),

                  ],

                });

                if (dbSelect2 === "back") break;

                if (!dbSelect2) throw new Error("Database name is required");

                const collectionName = await input({

                  message: "Enter collection name:",

                  validate: (value) =>

                    value.length > 0 || "Collection name is required",

                });

                if (!collectionName)

                  throw new Error("Collection name is required");

                await exportCollection(dbSelect2, collectionName);

                break;

              case "import-collection":

                break;

              case "create-db-image":

                const dbName = await input({

                  message: "Enter database name (type back to go back):",

                  validate: (value) =>

                    value.length > 0 || "Database name is required",

                  default: "webforma",

                });

                if (dbName === "back") break;

                await createDBImage(dbName);

                break;

              case "migrate-db":

                const sourceDB = await input({

                  message: "Enter source database name (type back to go back):",

                  validate: (value) =>

                    value.length > 0 || "Database name is required",

                });

                if (sourceDB === "back") break;

  

                const targetDB = await input({

                  message: "Enter target database name (type back to go back):",

                  validate: (value) =>

                    value.length > 0 || "Database name is required",

                });

  

                if (targetDB === "back") break;

                await migrateDB(sourceDB, targetDB);

                break;

            }

            break;

  

          case "build":

            const appChoice2 = await select({

              message: "Select an app to build:",

              choices: [

                { name: "<- Go Back", value: "back" },

                ...appList.map((app) => ({

                  name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                  value: app.name,

                })),

              ],

            });

            if (appChoice2 === "back") break;

            await buildApp(appChoice2);

            break;

          case "build-local-prod":

            const appChoice3 = await select({

              message: "Select an app to build:",

              choices: [

                { name: "<- Go Back", value: "back" },

                ...appList.map((app) => ({

                  name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                  value: app.name,

                })),

              ],

            });

            if (appChoice3 === "back") break;

            await runLocalProdBuild(appChoice3);

            break;

          case "connect-docker":

            const appChoice5 = await select({

              message: "Select an app to build:",

              choices: [

                { name: "<- Go Back", value: "back" },

                ...appList.map((app) => ({

                  name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                  value: app.name,

                })),

              ],

            });

            if (appChoice5 === "back") break;

  

            const appOrDb = await select({

              message: "Select an app or database to connect to:",

              choices: [

                { name: "App", value: "app" },

                { name: "Database", value: "db" },

              ],

            });

  

            const env = await select({

              message: "Select an environment:",

              choices: [

                { name: "Dev", value: "dev" },

                { name: "Test", value: "test" },

                { name: "Local Prod", value: "local-prod" },

                { name: "Prod", value: "prod" },

              ],

            });

  

            await connectToDocker(

              appChoice5,

              appOrDb,

              env,

              false,

              getDefaultLogOptions(),

            );

            break;

          case "deploy": // Handle the Deploy submenu

            const deployChoice = await select({

              message: "Select a deployment option:",

              choices: [

                { name: "<- Go Back", value: "back" },

                { name: "Deploy to Vercel", value: "deploy-vercel" },

                { name: "Deploy to Coolify", value: "deploy-coolify" },

                { name: "Deploy to VPS", value: "deploy-vps" },

              ],

            });

            if (deployChoice === "back") break; // Go back to the previous menu

            switch (deployChoice) {

              case "deploy-vercel":

                const vercelAppChoice = await select({

                  message: "Select an app to deploy to Vercel:",

                  choices: [

                    { name: "<- Go Back", value: "back" },

                    ...appList.map((app) => ({

                      name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (vercelAppChoice === "back") break;

                await deployToVercel(vercelAppChoice);

                break;

  

              case "deploy-coolify":

                const coolifyAppChoice = await select({

                  message: "Select an app to deploy to Coolify:",

                  choices: [

                    { name: "<- Go Back", value: "back" },

  

                    ...appList.map((app) => ({

                      name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                      value: app.name,

                    })),

                  ],

                });

                if (coolifyAppChoice === "back") break;

                await deployToCoolify(coolifyAppChoice);

                break;

              case "deploy-vps":

                const vpsIp = await input({

                  message: "Enter VPS IP address:",

                });

                const vpsUser = await input({

                  message: "Enter VPS username:",

                });

                const sshKey = await input({

                  message: "Enter path to SSH key (leave empty if not needed):",

                  default: "",

                });

  

                // Upload the script using scp

                const uploadCommand = `wsl scp ${sshKey ? `-i ${sshKey}` : ""} start-db.sh ${vpsUser}@${vpsIp}:/tmp/start-db.sh`;

                await executeCommand(uploadCommand);

  

                // Execute the script on the VPS

                const executeCommandStr = `wsl ssh ${sshKey ? `-i ${sshKey}` : ""} ${vpsUser}@${vpsIp} "bash /tmp/start-db.sh"`;

                await executeCommand(executeCommandStr);

                break;

            }

            break;

  

          case "test": // Handle the Deploy submenu

            const testingChoice = await select({

              message: "Select a testing option:",

              choices: [

                { name: "<- Go Back", value: "back" },

                { name: "Playwright e2e tests", value: "playwright-e2e" },

                {

                  name: "Playwright components tests",

                  value: "playwright-components",

                },

              ],

            });

            if (testingChoice === "back") break; // Go back to the previous menu

            // add scripts here

            if (testingChoice === "playwright-e2e") {

              const appName = await select({

                message: "Select an app to test:",

                choices: [

                  { name: "<- Go Back", value: "back" },

  

                  ...appList.map((app) => ({

                    name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                    value: app.name,

                  })),

                ],

              });

              if (appName === "back") break;

              console.log("Starting e2e tests for app: ", appName);

              await startTestServer(appName, "test");

              break;

            }

  

            if (testingChoice === "playwright-components") {

              break;

            }

  

            break;

  

          case "build-test-deploy": // Handle the Deploy submenu

            const buildTestDeployChoice = await select({

              message: "Select project to build test and deploy:",

  

              choices: [

                { name: "<- Go Back", value: "back" },

                { name: "Verify Status", value: "verify-status" },

                {

                  name: "--- All ---",

                  value: "all",

                },

                ...appList.map((app) => ({

                  name: `${app.name} - ${app.isTemplate ? "Template" : "App"} - (${app.path})`,

                  value: app.name,

                })),

              ],

            });

            if (buildTestDeployChoice === "back") break; // Go back to the previous menu

            if (buildTestDeployChoice === "verify-status") {

              const _buildNumber = await input({

                message: "Enter build number:",

                validate: (value) =>

                  !isNaN(Number(value)) || "Must be a number",

              });

  

              console.log(status);

              break;

            }

  

            await buildProcess(buildTestDeployChoice);

  

            break;

  

          case "misc": // Handle the Misc submenu

            const miscChoice = await select({

              message: "Select an option:",

              choices: [

                { name: "<- Go Back", value: "back" },

                {

                  name: "Check Next.js version",

                  value: "check-next-version",

                },

                {

                  name: "Post Plus Standalone",

                  value: "post-plus-standalone",

                },

                { name: "Post Pro Standalone", value: "post-pro-standalone" },

                { name: "Post Lite Export", value: "post-lite-export" },

                { name: "Upload to R2", value: "upload-r2" },

                {

                  name: "Download Relume components",

                  value: "download-relume",

                },

                { name: "TS ignore files", value: "ts-ignore-files" },

              ],

            });

            if (miscChoice === "back") break; // Go back to the previous menu

            switch (miscChoice) {

              case "check-next-version":

                await checkNextVersion();

                await checkPayloadVersion();

                break;

              case "post-plus-standalone":

                await copyStandaloneFiles({

                  projectName: "webforma-plus",

                  projectType: "templates",

                });

                break;

              case "post-pro-standalone":

                await copyStandaloneFiles({

                  projectName: "webforma-pro",

                  projectType: "templates",

                });

                break;

              case "post-lite-export":

                await copyStandaloneFiles({

                  projectName: "webforma-lite",

                  projectType: "templates",

                });

                break;

              case "upload-r2":

                const folderPath = await input({

                  message:

                    "Enter folder path to upload (type back to go back):",

                  validate: (value) =>

                    value.length > 0 || "Folder path is required",

                });

                if (folderPath === "back") break; // Go back to the previous menu

                await uploadToR2(folderPath, getConfig());

                break;

              case "ts-ignore-files":

                const tsIgnoreFolderPath = await input({

                  message: "Please provide the folder path:",

                  validate: (input) => {

                    // Validate that the input is not empty

                    return input.length > 0 || "Folder path is required";

                  },

                });

                if (tsIgnoreFolderPath === "back") break; // Go back to the previous menu

                // await tsIgnoreFiles(tsIgnoreFolderPath);

                break;

              case "download-relume":

                // Get the parameters from the user

                const startIndex = await input({

                  message: "Enter start index:",

                  validate: (value) =>

                    !isNaN(Number(value)) || "Must be a number",

                });

  

                const endIndex = await input({

                  message: "Enter end index:",

                  validate: (value) =>

                    !isNaN(Number(value)) || "Must be a number",

                });

  

                const sectionName = await input({

                  message: "Enter section name:",

                  validate: (value) =>

                    value.length > 0 || "Section name is required",

                });

  

                // Call the download function with the parameters

                await downloadRelumeComponents(

                  parseInt(startIndex),

                  parseInt(endIndex),

                  sectionName,

                );

                break;

            }

            break;

  

          // ... rest of the script choices ...

  

          case "exit":

            console.log(chalk.green("👋 Goodbye!"));

            process.exit(0);

            break;

  

          default:

            console.error(chalk.red("Invalid script selected"));

            continue;

        }

      } catch (error: unknown) {

        if (

          error instanceof Error &&

          error.message.includes("User force closed")

        ) {

          cleanup();

          return;

        }

        console.error(chalk.red("Error executing script:"), error);

        console.log(chalk.yellow("\n🛑 Returning to main menu..."));

        await new Promise((resolve) => setTimeout(resolve, 2000));

      }

    }

  } finally {

    // Remove listeners when function exits

    process.off("SIGINT", cleanup);

    process.off("SIGTERM", cleanup);

  }

}