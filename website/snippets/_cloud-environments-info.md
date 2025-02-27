## Types of environments

In dbt Cloud, there are two types of environments:
- **Deployment environment** &mdash; Determines the settings used when jobs created within that environment are executed.<br></br>
    Types of deployment environments:
    - General
    - Staging <Lifecycle status='beta' />
    - Production
- **Development environment** &mdash; Determines the settings used in the dbt Cloud IDE or dbt Cloud CLI, for that particular project. 

Each dbt Cloud project can only have a single development environment but can have any number of deployment environments.

|| Development  | Staging <b></b> <Lifecycle status='beta' /> | Deployment |
|------| --- | --- | --- |
| **Determines settings for** | dbt Cloud IDE or dbt Cloud CLI | dbt Cloud Job runs | dbt Cloud Job runs |
| **How many can I have in my project?** | 1 | Any number | Any number |

:::note 
For users familiar with development on dbt Core, each environment is roughly analogous to an entry in your `profiles.yml` file, with some additional information about your repository to ensure the proper version of code is executed. More info on dbt core environments [here](/docs/core/dbt-core-environments).
:::

## Common environment settings

Both development and deployment environments have a section called **General Settings**, which has some basic settings that all environments will define:

| Setting | Example Value | Definition | Accepted Values |
| --- | --- | --- | --- |
| Name | Production  | The environment name  | Any string! |
| Environment Type | Deployment | The type of environment | [Deployment, Development] |
| dbt Version | 1.4 (latest) | The dbt version used  | Any dbt version in the dropdown |
| Default to Custom Branch | ☑️ | Determines whether to use a branch other than the repository’s default  | See below |
| Custom Branch | dev | Custom Branch name | See below |

:::note About dbt version

- dbt Cloud allows users to select any dbt release. At this time, **environments must use a dbt version greater than or equal to v1.0.0;** [lower versions are no longer supported](/docs/dbt-versions/upgrade-dbt-version-in-cloud).
- If you select a current version with `(latest)` in the name, your environment will automatically install the latest stable version of the minor version selected.
- In 2024 we are introducing **Keep on latest version**, which removes the need for manually upgrading environments in the future, while ensuring you get access to the latest fixes and features. This feature is currently in beta for select customers, rolling out to wider availability through February and March._
:::

### Custom branch behavior

By default, all environments will use the default branch in your repository (usually the `main` branch) when accessing your dbt code. This is overridable within each dbt Cloud Environment using the **Default to a custom branch** option. This setting will have slightly different behavior depending on the environment type:

- **Development**: determines which branch in the dbt Cloud IDE or dbt Cloud CLI developers create branches from and open PRs against.
- **Deployment:** determines the branch is cloned during job executions for each environment.

For more info, check out this [FAQ page on this topic](/faqs/Environments/custom-branch-settings)!


### Extended attributes

:::note 
Extended attributes are retrieved and applied only at runtime when `profiles.yml` is requested for a specific Cloud run. Extended attributes are currently _not_ taken into consideration for SSH Tunneling which does not rely on `profiles.yml` values.
:::

Extended Attributes is a feature that allows users to set a flexible [profiles.yml](/docs/core/connect-data-platform/profiles.yml) snippet in their dbt Cloud Environment settings. It provides users with more control over environments (both deployment and development) and extends how dbt Cloud connects to the data platform within a given environment.

Extended Attributes is a text box extension at the environment level that overrides connection or environment credentials, including any custom environment variables. You can set any YAML attributes that a dbt adapter accepts in its `profiles.yml`.

Something to note, Extended Attributes don't mask secret values. We recommend avoiding setting secret values to prevent visibility in the text box and logs. 

<Lightbox src="/img/docs/dbt-cloud/using-dbt-cloud/extended-attributes.jpg" width="95%" title="Extended Attributes helps users add profiles.yml attributes to dbt Cloud Environment settings using a free form text box." /> <br />

If you're developing in the [dbt Cloud IDE](/docs/cloud/dbt-cloud-ide/develop-in-the-cloud), [dbt Cloud CLI](/docs/cloud/cloud-cli-installation), or [orchestrating job runs](/docs/deploy/deployments), Extended Attributes parses through the provided YAML and extracts the `profiles.yml` attributes. For each individual attribute:

- If the attribute exists in another source (such as your project settings), it will replace its value (like environment-level values) in the profile. It also overrides any custom environment variables.

- If the attribute doesn't exist, it will add the attribute or value pair to the profile. 

Only the **top-level keys** are accepted in extended attributes. This means that if you want to change a specific sub-key value, you must provide the entire top-level key as a JSON block in your resulting YAML. For example, if you want to customize a particular field within a [service account JSON](/docs/core/connect-data-platform/bigquery-setup#service-account-json) for your BigQuery connection (like 'project_id' or 'client_email'), you need to provide an override for the entire top-level `keyfile_json` main key/attribute using extended attributes. Include the sub-fields as a nested JSON block.

The following code is an example of the types of attributes you can add in the **Extended Attributes** text box:

```yaml
dbname: jaffle_shop      
schema: dbt_alice      
threads: 4
```

### Git repository caching 

At the start of every job run, dbt Cloud clones the project's Git repository so it has the latest versions of your project's code and runs `dbt deps` to install your dependencies. 

For improved reliability and performance on your job runs, you can enable dbt Cloud to keep a cache of the project's Git repository. So, if there's a third-party outage that causes the cloning operation to fail, dbt Cloud will instead use the cached copy of the repo so your jobs can continue running as scheduled. 

dbt Cloud caches your project's Git repo after each successful run and retains it for 8 days if there are no repo updates. It caches all packages regardless of installation method and does not fetch code outside of the job runs. 

dbt Cloud will use the cached copy of your project's Git repo under these circumstances:

- Outages from third-party services (for example, the [dbt package hub](https://hub.getdbt.com/)).
- Git authentication fails.
- There are syntax errors in the `packages.yml` file. You can set up and use [continuous integration (CI)](/docs/deploy/continuous-integration) to find these errors sooner.
- If a package doesn't work with the current dbt version. You can set up and use [continuous integration (CI)](/docs/deploy/continuous-integration) to identify this issue sooner.

To enable Git repository caching, select **Account settings** from the gear menu and enable the **Repository caching** option. 

<Lightbox src="/img/docs/deploy/example-account-settings.png" width="85%" title="Example of the Repository caching option" />

:::note

This feature is only available on the dbt Cloud Enterprise plan. 

:::

### Partial parsing

At the start of every dbt invocation, dbt reads all the files in your project, extracts information, and constructs an internal manifest containing every object (model, source, macro, and so on). Among other things, it uses the `ref()`, `source()`, and `config()` macro calls within models to set properties, infer dependencies, and construct your project's DAG. When dbt finishes parsing your project, it stores the internal manifest in a file called `partial_parse.msgpack`. 

Parsing projects can be time-consuming, especially for large projects with hundreds of models and thousands of files. To reduce the time it takes dbt to parse your project, use the partial parsing feature in dbt Cloud for your environment. When enabled, dbt Cloud uses the `partial_parse.msgpack` file to determine which files have changed (if any) since the project was last parsed, and then it parses _only_ the changed files and the files related to those changes.

Partial parsing in dbt Cloud requires dbt version 1.4 or newer. The feature does have some known limitations. Refer to [Known limitations](/reference/parsing#known-limitations) to learn more about them.

To enable, select **Account settings** from the gear menu and enable the **Partial parsing** option.

<Lightbox src="/img/docs/deploy/example-account-settings.png" width="85%" title="Example of the Partial parsing option" />
