---
sidebarDepth: 2
---

# Update to the latest Meilisearch version

Currently, Meilisearch databases are only compatible with the version of Meilisearch used to create them. The following guide will walk you through using a [dump](/learn/advanced/dumps.md) to migrate an existing database from an older version of Meilisearch to the most recent one.

If you're updating your Meilisearch instance on cloud platforms like DigitalOcean, AWS, or GCP, ensure that you can connect to your cloud instance via SSH. Depending on the user you are connecting with (root, admin, etc.), you may need to prefix some commands with `sudo`.

If migrating to the latest version of Meilisearch will cause you to skip multiple versions, this may require changes to your codebase. [Refer to our version-specific update warnings for more details](#version-specific-warnings).

::: tip
If you are running Meilisearch as a `systemctl` service using v0.22 or above, try our [migration script](https://github.com/meilisearch/meilisearch-migration).
:::

::: danger
This guide only works for versions v0.15 and above. If you are using an older version, please [contact support](https://discord.gg/meilisearch) for more information.
:::

## Step 1: Export data

### Verify your database version

First, verify the version of Meilisearch that's compatible with your database using the get version endpoint:

<CodeSamples id="updating_guide_check_version_new_authorization_header" />

The response should look something like this:

```json
{
  "commitSha": "stringOfLettersAndNumbers",
  "commitDate": "YYYY-MM-DDTimestamp",
  "pkgVersion": "x.y.z"
}
```

::: note
If you get the `missing_authorization_header` error, you might be using **v0.24 or below**. For each command, replace the `Authorization: Bearer` header with the `X-Meili-API-Key: API_KEY` header:

<CodeSamples id="updating_guide_check_version_old_authorization_header" />
:::

If your [`pkgVersion`](/reference/api/version.md#version-object) is 0.21 or above, you can jump to [creating the dump](#create-the-dump). If not, proceed to the next step.

### Set all fields as displayed attributes

::: note
If your dump was created in Meilisearch v0.21 or above, [skip this step](#create-the-dump).
:::

When creating dumps using Meilisearch versions below v0.21, all fields must be [displayed](/learn/configuration/displayed_searchable_attributes.md#displayed-fields) in order to be saved in the dump.

Start by verifying that all attributes are included in the displayed attributes list:

<CodeSamples id="updating_guide_get_displayed_attributes_old_authorization_header" />

If the response for all indexes is `{'displayedAttributes': '["*"]'}`, you can move on to the [next step](#create-the-dump).

If the response is anything else, save the current list of displayed attributes in a text file and then reset the displayed attributes list to its default value `(["*"])`:

<CodeSamples id="updating_guide_reset_displayed_attributes_old_authorization_header" />

This command returns an `updateId`. Use the get update endpoint to track the status of the operation:

```sh
 # replace {indexUid} with the uid of your index and {updateId} with the updateId returned by the previous request
  curl \
    -X GET 'http://<your-domain-name>/indexes/{indexUid}/updates/{updateId}'
    -H 'X-Meili-API-Key: API_KEY'
```

Once the status is `processed`, you're good to go. Repeat this process for all indexes, then move on to creating your dump.

### Create the dump

Before creating your dump, make sure that your [dump directory](/learn/configuration/instance_options.md#dump-directory) is somewhere accessible. By default, dumps are created in a folder called `dumps` at the root of your Meilisearch directory.

**Cloud platforms** like DigitalOcean, AWS, and GCP are configured to store dumps in the `/var/opt/meilisearch/dumps` directory.

If you're unsure where your Meilisearch directory is located, try this:

:::: tabs

::: tab UNIX

```bash
which meilisearch
```

It should return something like this:

```bash
/absolute/path/to/your/meilisearch/directory
```

:::

::: tab Windows CMD

```bash
where meilisearch
```

It should return something like this:

```bash
/absolute/path/to/your/meilisearch/directory
```

:::

::: tab Windows PowerShell

```bash
(Get-Command meilisearch).Path
```

It should return something like this:

```bash
/absolute/path/to/your/meilisearch/directory
```

:::

::::

::: danger `_geo` field in v0.27, v0.28, and v0.29
Due to an error allowing malformed `_geo` fields in Meilisearch **v0.27, v0.28, and v0.29**, you might not be able to import your dump. Please ensure the `_geo` field follows the [correct format](/learn/advanced/geosearch.md#preparing-documents-for-location-based-search) before creating your dump.
:::

You can then create a dump of your database:

<CodeSamples id="updating_guide_create_dump" />

The server should return a response that looks like this:

```json
{
  "taskUid": 1,
  "indexUid": null,
  "status": "enqueued",
  "type": "dumpCreation",
  "enqueuedAt": "2022-06-21T16:10:29.217688Z"
}
```

Use the `taskUid` to [track the status](/reference/api/tasks.md#get-one-task) of your dump. Keep in mind that the process can take some time to complete.

::: note
The response will vary slightly depending on your version. For v0.27 and below, the response returns a dump `uid`. You can track the status of the dump using the get dumps status endpoint:

```sh
  curl \
    -X GET 'http://<your-domain-name>/dumps/:dump_uid/status'
    -H 'Authorization: Bearer API_KEY' 
  # -H 'X-Meili-API-Key: API_KEY' for v0.24 or below
```

:::

Once the `dumpCreation` task shows `"status": "succeeded"`, you're ready to move on.

## Step 2: Prepare for migration

### Stop the Meilisearch instance

Stop your Meilisearch instance.

:::: tabs

::: tab Local installation

If you're running Meilisearch locally, you can stop the program with `Ctrl + c`.

:::

::: tab Cloud platforms

If you're running Meilisearch as a `systemctl` service, connect via SSH to your cloud instance and execute the following command to stop Meilisearch:

```bash
systemctl stop meilisearch
```

You may need to prefix the above command with `sudo` if you are not connected as root.

:::

::::

### Create a backup

Instead of deleting `data.ms`, we suggest creating a backup in case something goes wrong. `data.ms` should be at the root of the Meilisearch binary unless you chose [another location](/learn/configuration/instance_options.md#database-path).

On **cloud platforms**, you will find the `data.ms` folder at `/var/lib/meilisearch/data.ms`.

Move the binary of the current Meilisearch installation and database to the `/tmp` folder:

:::: tabs

::: tab Local installation

```
mv /path/to/your/meilisearch/directory/meilisearch/data.ms /tmp/
mv /path/to/your/meilisearch/directory/meilisearch /tmp/
```

:::

::: tab Cloud platforms

```
mv /usr/bin/meilisearch /tmp/
mv /var/lib/meilisearch/data.ms /tmp/
```

:::

::::

### Install the desired version of Meilisearch

Install the latest version of Meilisearch using:

:::: tabs

::: tab Local installation

```bash
curl -L https://install.meilisearch.com | sh
```

:::

::: tab Cloud platforms

```sh
# replace {meilisearch_version} with the version of your choice. Use the format: `vX.X.X`
curl "https://github.com/meilisearch/meilisearch/releases/download/{meilisearch_version}/meilisearch-linux-amd64" --output meilisearch --location --show-error
```

:::

::::

Give execute permission to the Meilisearch binary:

```
chmod +x meilisearch
```

For **cloud platforms**, move the new Meilisearch binary to the `/usr/bin` directory:

```
mv meilisearch /usr/bin/meilisearch
```

## Step 3: Import data

### Launch Meilisearch and import the dump

Execute the command below to import the dump at launch:

:::: tabs

::: tab Local installation

```bash
# replace {dump_uid.dump} with the name of your dump file
./meilisearch --import-dump dumps/{dump_uid.dump} --master-key="MASTER_KEY"
```

:::

::: tab Cloud platforms

```sh
# replace {dump_uid.dump} with the name of your dump file
meilisearch --db-path /var/lib/meilisearch/data.ms --import-dump "/var/opt/meilisearch/dumps/{dump_uid.dump}"
```

:::

::::

Importing a dump requires indexing all the documents it contains. Depending on the size of your dataset, this process can take a long time and cause a spike in memory usage.

### Restart Meilisearch as a service

If you're running a **cloud instance**, press `Ctrl`+`C`  to stop Meilisearch once your dump has been correctly imported. Next, execute the following command to run the script to configure Meilisearch and restart it as a service:

```
meilisearch-setup
```

If required, set `displayedAttributes` back to its previous value using the [update displayed attributes endpoint](/reference/api/settings.md#update-displayed-attributes).

## Conclusion

Now that your updated Meilisearch instance is up and running, verify that the dump import was successful and no data was lost.

If everything looks good, then congratulations! You successfully migrated your database to the latest version of Meilisearch. Be sure to check out the [changelogs](https://github.com/meilisearch/MeiliSearch/releases).

If something went wrong, you can always roll back to the previous version. Feel free to [reach out for help](https://discord.gg/meilisearch) if the problem continues. If you successfully migrated your database but are having problems with your codebase, be sure to check out our [version-specific warnings](#version-specific-warnings).

### Delete backup files or rollback (_optional_)

Delete the Meilisearch binary and `data.ms` folder created by the previous steps. Next, move the backup files back to their previous location using:

:::: tabs

::: tab Local installation

```
mv /tmp/meilisearch /path/to/your/meilisearch/directory/meilisearch
mv /tmp/data.ms /path/to/your/meilisearch/directory/meilisearch/data.ms
```

:::

::: tab Cloud platforms

```
mv /tmp/meilisearch /usr/bin/meilisearch
mv /tmp/data.ms /var/lib/meilisearch/data.ms
```

:::

::::

For **cloud platforms** run the configuration script at the root of your Meilisearch directory:

```
meilisearch-setup
```

If all went well, you can delete the backup files using:

```
rm -r /tmp/meilisearch
rm -r /tmp/data.ms
```

You can also delete the dump file if desired:

:::: tabs

::: tab Local installation

```
rm /path/to/your/meilisearch/directory/meilisearch/dumps/{dump_uid.dump}
```

:::

::: tab Cloud platforms

```
rm /var/opt/meilisearch/dumps/{dump_uid.dump}
```

:::

::::

## Version-specific warnings

After migrating to the most recent version of Meilisearch, your code-base may require some changes. This section contains warnings for some of the most impactful version-specific changes. For full changelogs, see the [releases tab on GitHub](https://github.com/meilisearch/meilisearch/releases).

- If you are updating from **v0.25 or below**, be aware that:
  - The `private` and `public` keys have been deprecated and replaced by two default API keys with similar permissions: `Default Admin API Key` and `Default Search API Key`.
  - The `updates` API has been replaced with the `tasks` API.
- If you are **updating from v0.27 or below**, existing keys will have their `key` and `uid` fields regenerated.
