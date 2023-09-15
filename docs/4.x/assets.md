---
containsGeneratedContent: yes
---

# Assets

Craft lets you manage media and document files (“assets”) just like entries and other content types. Assets can live anywhere—a directory on the web server, or a remote storage service like Amazon S3.

Assets are one of Craft’s built-in [element types](elements.md), and are represented throughout the application as instances of <craft4:craft\elements\Asset>.

## Volumes

Assets are organized into **volumes**, each of which sits on top of a [filesystem](#filesystems) and carries its own permissions and [content](#asset-custom-fields) options. Volumes are configured from **Settings** → **Assets**.

When setting up a volume, you will be asked to create its underlying filesystem, as well.

<BrowserShot
  url="https://my-craft-project.ddev.site/admin/assets/volumes/new"
  :link="false"
  caption="You can create a filesystem without leaving the volume screen.">
<img src="./images/assets-new-volume-fs.png" alt="Screenshot of the volume settings screen in Craft with a slide-out for filesystem settings">
</BrowserShot>

Volumes can store their [transforms](#image-transforms) alongside the original images, or in a separate filesystem altogether. This is useful for private volumes or filesystems that still benefit from having previews available in the control panel.

## Filesystems

Filesystems decouple asset management (organization, permissions, and content) from the minutiae of actually storing and serving files.

All filesystems support the following options:

- **Files in this filesystem have public URLs**: Whether Craft should bother to generate URLs for the assets.
- **Base URL**: The public URL to files in this filesystem.

::: tip
A filesystem’s **Base URL** can be set to an environment variable, or begin with an alias. [Read more](./config/README.md#control-panel-settings) about special configuration values.

In the screenshot above, we’re using a `@cdn` [alias](./config/README.md#aliases) so that the URL can be updated across multiple filesystems with a single change.
:::

### Local Filesystems

Out of the box, the only type of filesystem Craft supports is a “Local” directory, on the same server. Local filesystems have one additional setting:
- **Base Path**: Set the filesystem’s root directory on the server. This must be within your web root in order for public URLs to work.

::: tip
The **Base Path** can be set to an environment variable or begin with an alias, just like the **Base URL**. Declaring both a `@web` and `@webroot` alias can simplify the process of configuring local filesystems.
:::

Craft/PHP must be able to write to any directories you use for a local filesystem.

### Remote Volumes

If you would prefer to store your assets on a remote storage service like Amazon S3, you can install a plugin that provides the appropriate filesystem adapter.

- [Amazon S3](https://github.com/craftcms/aws-s3) (first party)
- [Google Cloud Storage](https://github.com/craftcms/google-cloud) (first party)
- [Rackspace Cloud Files](https://github.com/craftcms/rackspace) (first party)
- [DigitalOcean Spaces](https://github.com/vaersaagod/dospaces) (Værsågod)
- [fortrabbit Object Storage](https://github.com/fortrabbit/craft-object-storage) (fortrabbit)

The settings for each type of filesystem will differ based on the provider, and may involve secrets. We recommend using [special config values](./config/README.md#control-panel-settings) to store and use these, securely.

## Asset Custom Fields

Each volume has its own [field layout](./fields.md#field-layouts), configured on its setting screen under the **Field Layout** tab.

### `alt` Text

Asset field layouts can include the native **Alternative Text** <Poi label="1" target="assetFieldLayout" id="altText" /> field layout element:

<BrowserShot
  url="https://my-craft-project.ddev.site/admin/assets/volumes/1"
  :link="false"
  id="assetFieldLayout"
  :poi="{
    altText: [90, 32.5],
  }">
<img src="./images/assets-field-layout.png" />
</BrowserShot>

Craft 4 introduced the `alt` attribute to standardize the inclusion of assistive text on `img` elements that Craft generates—especially in the control panel. Alt text is also added when outputting an image with `asset.getImg()` in Twig. You can always render `img` elements yourself, using any [custom field](./fields.md) values, attributes, or combination thereof. 

We strongly recommend adding the native attribute to your volumes’ field layouts; alt text is a critical interface for many users, and essential for anyone using assistive technology in the control panel. Well-considered image descriptions (and titles!) have the added benefit of making search and discovery of previously-uploaded images much easier. The WCAG [advises against](https://www.w3.org/TR/2015/REC-ATAG20-20150924/Overview.html#gl_b23) automatically repairing alt text with “generic [or] irrelevant strings,” including the name of the file (which asset titles are generated from), so Craft omits the `alt` attribute when using `asset.getImg()` if no explicit text is available.

**Alternative Text** is also displayed as a “transcript” beneath video previews, in the control panel.

::: tip
Do you have existing `alt` text stored in a different field? You can migrate it to the native attribute with the [`resave/assets` command](./console-commands.md#resave):

```bash
php craft resave/assets --set alt --to myAltTextField --if-empty
```
:::

## Assets Page

After creating your first volume, an **Assets** item will be added to the main control panel navigation. Clicking on it will take you to the Assets page, which shows a list of all of your volumes in the left sidebar, and the selected volume’s files and subfolders in the main content area.

In addition to the normal actions available in [element indexes](./elements.md#indexes), asset indexes support:

- Uploading new files using the **Upload files** toolbar button or by dragging files from your desktop;
- Creating and organizing [folders](#managing-subfolders) within a volume;
- Transferring a file from one volume to another by dragging-and-dropping it from the element index into a folder in the sources sidebar (or using the **Move…** element action);

Special [element actions](./elements.md#actions) are also available for single assets:

- Rename an existing file;
- Replace a file with a new one;
- Open the [image editor](#image-editor) (images only);
- Preview a file;
- Copy a public URL to a file;
- Copy a reference tag to a file;
- Move the selected assets to a new volume and/or folder;

### Managing Subfolders

::: tip
Asset and folder management was [greatly enhanced](https://craftcms.com/blog/craft-4-4-released) in Craft 4.4. Earlier versions only support drag-and-drop file management, and folders were created and deleted via the sources sidebar.
:::

<BrowserShot
  url="https://my-craft-project.ddev.site/admin/assets/uploads"
  :link="false"
  id="assetIndex"
  :poi="{
    breadcrumbs: [42, 21],
    folder: [57, 34],
    dragging: [46, 42],
    actions: [67, 94],
  }">
<img src="./images/assets-index.png" alt="Asset element index showing subfolder creation and drag-and-drop interface for organizing files" />
</BrowserShot>

Volumes are initialized with only a root folder, indicated by a “home” icon in the breadcrumbs. Subfolders can be created by clicking the caret <Poi label="1" target="assetIndex" id="breadcrumbs" /> next to the current folder.

The new subfolder will appear in-line <Poi label="2" target="assetIndex" id="folder" /> with the assets in the current folder. Assets and folders can be moved in a few different ways:

- Drag-and-drop <Poi label="3" target="assetIndex" id="dragging" /> one or more assets onto a folder in the table or thumbnail view _or_ onto a breadcrumb <Poi label="1" target="assetIndex" id="breadcrumbs" /> segment;
- Select assets with the checkboxes in each row, choose **Move…** from the actions <Poi label="4" target="assetIndex" id="actions" /> menu, and pick a destination folder;
- An entire folder can also be moved using the caret next to its breadcrumb;

The first method is a great way to quickly move assets into a parent directory, or back to the volume’s root folder.

::: tip
You can automatically organize assets when they are uploaded via an [assets field](./assets-fields.md) with the **Restrict assets to a single location** setting.
:::

## Updating Asset Indexes

If any files are ever added, modified, or deleted outside of Craft (such as over FTP), you’ll need to tell Craft to update its indexes for the volume. You can do that from **Utilities** → **Asset Indexes**.

You will have the option to cache remote images. If you don’t have any remote volumes (Amazon S3, etc.), you can safely ignore it. Enabling the setting will cause the indexing process to take longer to complete, but it will improve the speed of [image transform](image-transforms.md) generation.

## Image Transforms

Craft provides a way to perform a variety of image transformations to your assets. See [Image Transforms](image-transforms.md) for more information.

## Image Editor

Craft provides a built-in Image Editor for making changes to your images. You can crop, straighten, rotate, and flip your images, as well as choose a focal point on them.

To launch the Image Editor, double-click an image (either on the Assets page or from an [Assets field](assets-fields.md)) and press **Edit** in the top-right of the image preview area in the HUD. Alternatively, you can select an asset on the [Assets page](#assets-page) and choose **Edit image** from the task menu (<icon kind="settings" />).

### Focal Points

Set focal points on your images so Craft knows which part of the image to prioritize when determining how to crop your images for [image transforms](image-transforms.md). Focal points take precedence over the transform’s Crop Position setting.

To set a focal point, open the Image Editor and click on the Focal Point button. A circular icon will appear in the center of your image. Drag it to wherever you want the image’s focal point to be.

To remove the focal point, click on the Focal Point button again.

Like other changes in the Image Editor, focal points won’t take effect until you’ve saved the image.

## Querying Assets

You can fetch assets in your templates or PHP code using **asset queries**.

::: code
```twig
{# Create a new asset query #}
{% set myAssetQuery = craft.assets() %}
```
```php
// Create a new asset query
$myAssetQuery = \craft\elements\Asset::find();
```
:::

Once you’ve created an asset query, you can set [parameters](#parameters) on it to narrow down the results, and then [execute it](element-queries.md#executing-element-queries) by calling `.all()`. An array of [Asset](craft4:craft\elements\Asset) objects will be returned.

::: tip
See [Element Queries](element-queries.md) to learn about how element queries work.
:::

### Example

We can display a list of thumbnails for images in a “Photos” volume by doing the following:

1. Create an asset query with `craft.assets()`.
2. Set the [volume](#volume) and [kind](#kind) parameters on it.
3. Fetch the assets with `.all()`.
4. Loop through the assets using a [for](https://twig.symfony.com/doc/3.x/tags/for.html) tag to create the thumbnail list HTML.

```twig
{# Create an asset query with the 'volume' and 'kind' parameters #}
{% set myAssetQuery = craft.assets()
  .volume('photos')
  .kind('image') %}

{# Fetch the assets #}
{% set images = myAssetQuery.all() %}

{# Display the thumbnail list #}
<ul>
  {% for image in images %}
    <li><img src="{{ image.getUrl('thumb') }}" alt="{{ image.alt }}"></li>
  {% endfor %}
</ul>
```

::: warning
When using `asset.url` or `asset.getUrl()`, the asset’s source volume must have “Assets in this volume have public URLs” enabled and a “Base URL” setting. Otherwise, the result will always be empty.
:::

You can cache-bust asset URLs automatically by enabling the [revAssetUrls](config4:revAssetUrls) config setting, or handle them individually by using Craft’s [`url()` function](dev/functions.md#url) to append a query parameter with the last-modified timestamp:

```twig
<img src="{{ url(image.getUrl('thumb'), {v: image.dateModified.timestamp}) }}">
{# <img src="https://my-project.tld/images/_thumb/bar.jpg?v=1614374621"> #}
```

### Parameters

Asset queries support the following parameters:

<!-- BEGIN PARAMS -->



<!-- textlint-disable -->

| Param                                     | Description
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| [afterPopulate](#afterpopulate)           | Performs any post-population processing on elements.
| [andRelatedTo](#andrelatedto)             | Narrows the query results to only assets that are related to certain other elements.
| [asArray](#asarray)                       | Causes the query to return matching assets as arrays of data, rather than [Asset](craft4:craft\elements\Asset) objects.
| [cache](#cache)                           | Enables query cache for this Query.
| [clearCachedResult](#clearcachedresult)   | Clears the [cached result](https://craftcms.com/docs/4.x/element-queries.html#cache).
| [collect](#collect)                       |
| [dateCreated](#datecreated)               | Narrows the query results based on the assets’ creation dates.
| [dateModified](#datemodified)             | Narrows the query results based on the assets’ files’ last-modified dates.
| [dateUpdated](#dateupdated)               | Narrows the query results based on the assets’ last-updated dates.
| [filename](#filename)                     | Narrows the query results based on the assets’ filenames.
| [fixedOrder](#fixedorder)                 | Causes the query results to be returned in the order specified by [id](#id).
| [folderId](#folderid)                     | Narrows the query results based on the folders the assets belong to, per the folders’ IDs.
| [folderPath](#folderpath)                 | Narrows the query results based on the folders the assets belong to, per the folders’ paths.
| [hasAlt](#hasalt)                         | Narrows the query results based on whether the assets have alternative text.
| [height](#height)                         | Narrows the query results based on the assets’ image heights.
| [id](#id)                                 | Narrows the query results based on the assets’ IDs.
| [ignorePlaceholders](#ignoreplaceholders) | Causes the query to return matching assets as they are stored in the database, ignoring matching placeholder elements that were set by [craft\services\Elements::setPlaceholderElement()](https://docs.craftcms.com/api/v3/craft-services-elements.html#method-setplaceholderelement).
| [inReverse](#inreverse)                   | Causes the query results to be returned in reverse order.
| [includeSubfolders](#includesubfolders)   | Broadens the query results to include assets from any of the subfolders of the folder specified by [folderId](#folderid).
| [kind](#kind)                             | Narrows the query results based on the assets’ file kinds.
| [limit](#limit)                           | Determines the number of assets that should be returned.
| [offset](#offset)                         | Determines how many assets should be skipped in the results.
| [orderBy](#orderby)                       | Determines the order that the assets should be returned in. (If empty, defaults to `dateCreated DESC`.)
| [preferSites](#prefersites)               | If [unique](#unique) is set, this determines which site should be selected when querying multi-site elements.
| [prepareSubquery](#preparesubquery)       | Prepares the element query and returns its subquery (which determines what elements will be returned).
| [relatedTo](#relatedto)                   | Narrows the query results to only assets that are related to certain other elements.
| [savable](#savable)                       | Sets the [savable](https://docs.craftcms.com/api/v3/craft-elements-db-assetquery.html#savable) property.
| [search](#search)                         | Narrows the query results to only assets that match a search query.
| [site](#site)                             | Determines which site(s) the assets should be queried in.
| [siteId](#siteid)                         | Determines which site(s) the assets should be queried in, per the site’s ID.
| [siteSettingsId](#sitesettingsid)         | Narrows the query results based on the assets’ IDs in the `elements_sites` table.
| [size](#size)                             | Narrows the query results based on the assets’ file sizes (in bytes).
| [title](#title)                           | Narrows the query results based on the assets’ titles.
| [trashed](#trashed)                       | Narrows the query results to only assets that have been soft-deleted.
| [uid](#uid)                               | Narrows the query results based on the assets’ UIDs.
| [unique](#unique)                         | Determines whether only elements with unique IDs should be returned by the query.
| [uploader](#uploader)                     | Narrows the query results based on the user the assets were uploaded by, per the user’s IDs.
| [volume](#volume)                         | Narrows the query results based on the volume the assets belong to.
| [volumeId](#volumeid)                     | Narrows the query results based on the volumes the assets belong to, per the volumes’ IDs.
| [width](#width)                           | Narrows the query results based on the assets’ image widths.
| [with](#with)                             | Causes the query to return matching assets eager-loaded with related elements.
| [withTransforms](#withtransforms)         | Causes the query to return matching assets eager-loaded with image transform indexes.


<!-- textlint-enable -->


#### `afterPopulate`

Performs any post-population processing on elements.










#### `andRelatedTo`

Narrows the query results to only assets that are related to certain other elements.



See [Relations](https://craftcms.com/docs/4.x/relations.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch all assets that are related to myCategoryA and myCategoryB #}
{% set assets = craft.assets()
  .relatedTo(myCategoryA)
  .andRelatedTo(myCategoryB)
  .all() %}
```

```php
// Fetch all assets that are related to $myCategoryA and $myCategoryB
$assets = \craft\elements\Asset::find()
    ->relatedTo($myCategoryA)
    ->andRelatedTo($myCategoryB)
    ->all();
```
:::


#### `asArray`

Causes the query to return matching assets as arrays of data, rather than [Asset](craft4:craft\elements\Asset) objects.





::: code
```twig
{# Fetch assets as arrays #}
{% set assets = craft.assets()
  .asArray()
  .all() %}
```

```php
// Fetch assets as arrays
$assets = \craft\elements\Asset::find()
    ->asArray()
    ->all();
```
:::


#### `cache`

Enables query cache for this Query.










#### `clearCachedResult`

Clears the [cached result](https://craftcms.com/docs/4.x/element-queries.html#cache).






#### `collect`








#### `dateCreated`

Narrows the query results based on the assets’ creation dates.



Possible values include:

| Value | Fetches assets…
| - | -
| `'>= 2018-04-01'` | that were created on or after 2018-04-01.
| `'< 2018-05-01'` | that were created before 2018-05-01.
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were created between 2018-04-01 and 2018-05-01.
| `now`/`today`/`tomorrow`/`yesterday` | that were created at midnight of the specified relative date.



::: code
```twig
{# Fetch assets created last month #}
{% set start = date('first day of last month')|atom %}
{% set end = date('first day of this month')|atom %}

{% set assets = craft.assets()
  .dateCreated(['and', ">= #{start}", "< #{end}"])
  .all() %}
```

```php
// Fetch assets created last month
$start = (new \DateTime('first day of last month'))->format(\DateTime::ATOM);
$end = (new \DateTime('first day of this month'))->format(\DateTime::ATOM);

$assets = \craft\elements\Asset::find()
    ->dateCreated(['and', ">= {$start}", "< {$end}"])
    ->all();
```
:::


#### `dateModified`

Narrows the query results based on the assets’ files’ last-modified dates.

Possible values include:

| Value | Fetches assets…
| - | -
| `'>= 2018-04-01'` | that were modified on or after 2018-04-01.
| `'< 2018-05-01'` | that were modified before 2018-05-01.
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were modified between 2018-04-01 and 2018-05-01.
| `now`/`today`/`tomorrow`/`yesterday` | that were modified at midnight of the specified relative date.



::: code
```twig
{# Fetch assets modified in the last month #}
{% set start = date('30 days ago')|atom %}

{% set assets = craft.assets()
  .dateModified(">= #{start}")
  .all() %}
```

```php
// Fetch assets modified in the last month
$start = (new \DateTime('30 days ago'))->format(\DateTime::ATOM);

$assets = \craft\elements\Asset::find()
    ->dateModified(">= {$start}")
    ->all();
```
:::


#### `dateUpdated`

Narrows the query results based on the assets’ last-updated dates.



Possible values include:

| Value | Fetches assets…
| - | -
| `'>= 2018-04-01'` | that were updated on or after 2018-04-01.
| `'< 2018-05-01'` | that were updated before 2018-05-01.
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were updated between 2018-04-01 and 2018-05-01.
| `now`/`today`/`tomorrow`/`yesterday` | that were updated at midnight of the specified relative date.



::: code
```twig
{# Fetch assets updated in the last week #}
{% set lastWeek = date('1 week ago')|atom %}

{% set assets = craft.assets()
  .dateUpdated(">= #{lastWeek}")
  .all() %}
```

```php
// Fetch assets updated in the last week
$lastWeek = (new \DateTime('1 week ago'))->format(\DateTime::ATOM);

$assets = \craft\elements\Asset::find()
    ->dateUpdated(">= {$lastWeek}")
    ->all();
```
:::


#### `filename`

Narrows the query results based on the assets’ filenames.

Possible values include:

| Value | Fetches assets…
| - | -
| `'foo.jpg'` | with a filename of `foo.jpg`.
| `'foo*'` | with a filename that begins with `foo`.
| `'*.jpg'` | with a filename that ends with `.jpg`.
| `'*foo*'` | with a filename that contains `foo`.
| `'not *foo*'` | with a filename that doesn’t contain `foo`.
| `['*foo*', '*bar*']` | with a filename that contains `foo` or `bar`.
| `['not', '*foo*', '*bar*']` | with a filename that doesn’t contain `foo` or `bar`.



::: code
```twig
{# Fetch all the hi-res images #}
{% set assets = craft.assets()
  .filename('*@2x*')
  .all() %}
```

```php
// Fetch all the hi-res images
$assets = \craft\elements\Asset::find()
    ->filename('*@2x*')
    ->all();
```
:::


#### `fixedOrder`

Causes the query results to be returned in the order specified by [id](#id).



::: tip
If no IDs were passed to [id](#id), setting this to `true` will result in an empty result set.
:::



::: code
```twig
{# Fetch assets in a specific order #}
{% set assets = craft.assets()
  .id([1, 2, 3, 4, 5])
  .fixedOrder()
  .all() %}
```

```php
// Fetch assets in a specific order
$assets = \craft\elements\Asset::find()
    ->id([1, 2, 3, 4, 5])
    ->fixedOrder()
    ->all();
```
:::


#### `folderId`

Narrows the query results based on the folders the assets belong to, per the folders’ IDs.

Possible values include:

| Value | Fetches assets…
| - | -
| `1` | in a folder with an ID of 1.
| `'not 1'` | not in a folder with an ID of 1.
| `[1, 2]` | in a folder with an ID of 1 or 2.
| `['not', 1, 2]` | not in a folder with an ID of 1 or 2.



::: code
```twig
{# Fetch assets in the folder with an ID of 1 #}
{% set assets = craft.assets()
  .folderId(1)
  .all() %}
```

```php
// Fetch assets in the folder with an ID of 1
$assets = \craft\elements\Asset::find()
    ->folderId(1)
    ->all();
```
:::



::: tip
This can be combined with [includeSubfolders](#includesubfolders) if you want to include assets in all the subfolders of a certain folder.
:::
#### `folderPath`

Narrows the query results based on the folders the assets belong to, per the folders’ paths.

Possible values include:

| Value | Fetches assets…
| - | -
| `foo/` | in a `foo/` folder (excluding nested folders).
| `foo/*` | in a `foo/` folder (including nested folders).
| `'not foo/*'` | not in a `foo/` folder (including nested folders).
| `['foo/*', 'bar/*']` | in a `foo/` or `bar/` folder (including nested folders).
| `['not', 'foo/*', 'bar/*']` | not in a `foo/` or `bar/` folder (including nested folders).



::: code
```twig
{# Fetch assets in the foo/ folder or its nested folders #}
{% set assets = craft.assets()
  .folderPath('foo/*')
  .all() %}
```

```php
// Fetch assets in the foo/ folder or its nested folders
$assets = \craft\elements\Asset::find()
    ->folderPath('foo/*')
    ->all();
```
:::


#### `hasAlt`

Narrows the query results based on whether the assets have alternative text.






#### `height`

Narrows the query results based on the assets’ image heights.

Possible values include:

| Value | Fetches assets…
| - | -
| `100` | with a height of 100.
| `'>= 100'` | with a height of at least 100.
| `['>= 100', '<= 1000']` | with a height between 100 and 1,000.



::: code
```twig
{# Fetch XL images #}
{% set assets = craft.assets()
  .kind('image')
  .height('>= 1000')
  .all() %}
```

```php
// Fetch XL images
$assets = \craft\elements\Asset::find()
    ->kind('image')
    ->height('>= 1000')
    ->all();
```
:::


#### `id`

Narrows the query results based on the assets’ IDs.



Possible values include:

| Value | Fetches assets…
| - | -
| `1` | with an ID of 1.
| `'not 1'` | not with an ID of 1.
| `[1, 2]` | with an ID of 1 or 2.
| `['not', 1, 2]` | not with an ID of 1 or 2.



::: code
```twig
{# Fetch the asset by its ID #}
{% set asset = craft.assets()
  .id(1)
  .one() %}
```

```php
// Fetch the asset by its ID
$asset = \craft\elements\Asset::find()
    ->id(1)
    ->one();
```
:::



::: tip
This can be combined with [fixedOrder](#fixedorder) if you want the results to be returned in a specific order.
:::


#### `ignorePlaceholders`

Causes the query to return matching assets as they are stored in the database, ignoring matching placeholder
elements that were set by [craft\services\Elements::setPlaceholderElement()](https://docs.craftcms.com/api/v3/craft-services-elements.html#method-setplaceholderelement).










#### `inReverse`

Causes the query results to be returned in reverse order.





::: code
```twig
{# Fetch assets in reverse #}
{% set assets = craft.assets()
  .inReverse()
  .all() %}
```

```php
// Fetch assets in reverse
$assets = \craft\elements\Asset::find()
    ->inReverse()
    ->all();
```
:::


#### `includeSubfolders`

Broadens the query results to include assets from any of the subfolders of the folder specified by [folderId](#folderid).



::: code
```twig
{# Fetch assets in the folder with an ID of 1 (including its subfolders) #}
{% set assets = craft.assets()
  .folderId(1)
  .includeSubfolders()
  .all() %}
```

```php
// Fetch assets in the folder with an ID of 1 (including its subfolders)
$assets = \craft\elements\Asset::find()
    ->folderId(1)
    ->includeSubfolders()
    ->all();
```
:::



::: warning
This will only work if [folderId](#folderid) was set to a single folder ID.
:::
#### `kind`

Narrows the query results based on the assets’ file kinds.

Supported file kinds:
- `access`
- `audio`
- `compressed`
- `excel`
- `flash`
- `html`
- `illustrator`
- `image`
- `javascript`
- `json`
- `pdf`
- `photoshop`
- `php`
- `powerpoint`
- `text`
- `video`
- `word`
- `xml`
- `unknown`

Possible values include:

| Value | Fetches assets…
| - | -
| `'image'` | with a file kind of `image`.
| `'not image'` | not with a file kind of `image`..
| `['image', 'pdf']` | with a file kind of `image` or `pdf`.
| `['not', 'image', 'pdf']` | not with a file kind of `image` or `pdf`.



::: code
```twig
{# Fetch all the images #}
{% set assets = craft.assets()
  .kind('image')
  .all() %}
```

```php
// Fetch all the images
$assets = \craft\elements\Asset::find()
    ->kind('image')
    ->all();
```
:::


#### `limit`

Determines the number of assets that should be returned.



::: code
```twig
{# Fetch up to 10 assets  #}
{% set assets = craft.assets()
  .limit(10)
  .all() %}
```

```php
// Fetch up to 10 assets
$assets = \craft\elements\Asset::find()
    ->limit(10)
    ->all();
```
:::


#### `offset`

Determines how many assets should be skipped in the results.



::: code
```twig
{# Fetch all assets except for the first 3 #}
{% set assets = craft.assets()
  .offset(3)
  .all() %}
```

```php
// Fetch all assets except for the first 3
$assets = \craft\elements\Asset::find()
    ->offset(3)
    ->all();
```
:::


#### `orderBy`

Determines the order that the assets should be returned in. (If empty, defaults to `dateCreated DESC`.)



::: code
```twig
{# Fetch all assets in order of date created #}
{% set assets = craft.assets()
  .orderBy('dateCreated ASC')
  .all() %}
```

```php
// Fetch all assets in order of date created
$assets = \craft\elements\Asset::find()
    ->orderBy('dateCreated ASC')
    ->all();
```
:::


#### `preferSites`

If [unique](#unique) is set, this determines which site should be selected when querying multi-site elements.



For example, if element “Foo” exists in Site A and Site B, and element “Bar” exists in Site B and Site C,
and this is set to `['c', 'b', 'a']`, then Foo will be returned for Site B, and Bar will be returned
for Site C.

If this isn’t set, then preference goes to the current site.



::: code
```twig
{# Fetch unique assets from Site A, or Site B if they don’t exist in Site A #}
{% set assets = craft.assets()
  .site('*')
  .unique()
  .preferSites(['a', 'b'])
  .all() %}
```

```php
// Fetch unique assets from Site A, or Site B if they don’t exist in Site A
$assets = \craft\elements\Asset::find()
    ->site('*')
    ->unique()
    ->preferSites(['a', 'b'])
    ->all();
```
:::


#### `prepareSubquery`

Prepares the element query and returns its subquery (which determines what elements will be returned).






#### `relatedTo`

Narrows the query results to only assets that are related to certain other elements.



See [Relations](https://craftcms.com/docs/4.x/relations.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch all assets that are related to myCategory #}
{% set assets = craft.assets()
  .relatedTo(myCategory)
  .all() %}
```

```php
// Fetch all assets that are related to $myCategory
$assets = \craft\elements\Asset::find()
    ->relatedTo($myCategory)
    ->all();
```
:::


#### `savable`

Sets the [savable](https://docs.craftcms.com/api/v3/craft-elements-db-assetquery.html#savable) property.






#### `search`

Narrows the query results to only assets that match a search query.



See [Searching](https://craftcms.com/docs/4.x/searching.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Get the search query from the 'q' query string param #}
{% set searchQuery = craft.app.request.getQueryParam('q') %}

{# Fetch all assets that match the search query #}
{% set assets = craft.assets()
  .search(searchQuery)
  .all() %}
```

```php
// Get the search query from the 'q' query string param
$searchQuery = \Craft::$app->request->getQueryParam('q');

// Fetch all assets that match the search query
$assets = \craft\elements\Asset::find()
    ->search($searchQuery)
    ->all();
```
:::


#### `site`

Determines which site(s) the assets should be queried in.



The current site will be used by default.

Possible values include:

| Value | Fetches assets…
| - | -
| `'foo'` | from the site with a handle of `foo`.
| `['foo', 'bar']` | from a site with a handle of `foo` or `bar`.
| `['not', 'foo', 'bar']` | not in a site with a handle of `foo` or `bar`.
| a [craft\models\Site](craft4:craft\models\Site) object | from the site represented by the object.
| `'*'` | from any site.

::: tip
If multiple sites are specified, elements that belong to multiple sites will be returned multiple times. If you
only want unique elements to be returned, use [unique](#unique) in conjunction with this.
:::



::: code
```twig
{# Fetch assets from the Foo site #}
{% set assets = craft.assets()
  .site('foo')
  .all() %}
```

```php
// Fetch assets from the Foo site
$assets = \craft\elements\Asset::find()
    ->site('foo')
    ->all();
```
:::


#### `siteId`

Determines which site(s) the assets should be queried in, per the site’s ID.



The current site will be used by default.

Possible values include:

| Value | Fetches assets…
| - | -
| `1` | from the site with an ID of `1`.
| `[1, 2]` | from a site with an ID of `1` or `2`.
| `['not', 1, 2]` | not in a site with an ID of `1` or `2`.
| `'*'` | from any site.



::: code
```twig
{# Fetch assets from the site with an ID of 1 #}
{% set assets = craft.assets()
  .siteId(1)
  .all() %}
```

```php
// Fetch assets from the site with an ID of 1
$assets = \craft\elements\Asset::find()
    ->siteId(1)
    ->all();
```
:::


#### `siteSettingsId`

Narrows the query results based on the assets’ IDs in the `elements_sites` table.



Possible values include:

| Value | Fetches assets…
| - | -
| `1` | with an `elements_sites` ID of 1.
| `'not 1'` | not with an `elements_sites` ID of 1.
| `[1, 2]` | with an `elements_sites` ID of 1 or 2.
| `['not', 1, 2]` | not with an `elements_sites` ID of 1 or 2.



::: code
```twig
{# Fetch the asset by its ID in the elements_sites table #}
{% set asset = craft.assets()
  .siteSettingsId(1)
  .one() %}
```

```php
// Fetch the asset by its ID in the elements_sites table
$asset = \craft\elements\Asset::find()
    ->siteSettingsId(1)
    ->one();
```
:::


#### `size`

Narrows the query results based on the assets’ file sizes (in bytes).

Possible values include:

| Value | Fetches assets…
| - | -
| `1000` | with a size of 1,000 bytes (1KB).
| `'< 1000000'` | with a size of less than 1,000,000 bytes (1MB).
| `['>= 1000', '< 1000000']` | with a size between 1KB and 1MB.



::: code
```twig
{# Fetch assets that are smaller than 1KB #}
{% set assets = craft.assets()
  .size('< 1000')
  .all() %}
```

```php
// Fetch assets that are smaller than 1KB
$assets = \craft\elements\Asset::find()
    ->size('< 1000')
    ->all();
```
:::


#### `title`

Narrows the query results based on the assets’ titles.



Possible values include:

| Value | Fetches assets…
| - | -
| `'Foo'` | with a title of `Foo`.
| `'Foo*'` | with a title that begins with `Foo`.
| `'*Foo'` | with a title that ends with `Foo`.
| `'*Foo*'` | with a title that contains `Foo`.
| `'not *Foo*'` | with a title that doesn’t contain `Foo`.
| `['*Foo*', '*Bar*']` | with a title that contains `Foo` or `Bar`.
| `['not', '*Foo*', '*Bar*']` | with a title that doesn’t contain `Foo` or `Bar`.



::: code
```twig
{# Fetch assets with a title that contains "Foo" #}
{% set assets = craft.assets()
  .title('*Foo*')
  .all() %}
```

```php
// Fetch assets with a title that contains "Foo"
$assets = \craft\elements\Asset::find()
    ->title('*Foo*')
    ->all();
```
:::


#### `trashed`

Narrows the query results to only assets that have been soft-deleted.





::: code
```twig
{# Fetch trashed assets #}
{% set assets = craft.assets()
  .trashed()
  .all() %}
```

```php
// Fetch trashed assets
$assets = \craft\elements\Asset::find()
    ->trashed()
    ->all();
```
:::


#### `uid`

Narrows the query results based on the assets’ UIDs.





::: code
```twig
{# Fetch the asset by its UID #}
{% set asset = craft.assets()
  .uid('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx')
  .one() %}
```

```php
// Fetch the asset by its UID
$asset = \craft\elements\Asset::find()
    ->uid('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx')
    ->one();
```
:::


#### `unique`

Determines whether only elements with unique IDs should be returned by the query.



This should be used when querying elements from multiple sites at the same time, if “duplicate” results is not
desired.



::: code
```twig
{# Fetch unique assets across all sites #}
{% set assets = craft.assets()
  .site('*')
  .unique()
  .all() %}
```

```php
// Fetch unique assets across all sites
$assets = \craft\elements\Asset::find()
    ->site('*')
    ->unique()
    ->all();
```
:::


#### `uploader`

Narrows the query results based on the user the assets were uploaded by, per the user’s IDs.

Possible values include:

| Value | Fetches assets…
| - | -
| `1` | uploaded by the user with an ID of 1.
| a [craft\elements\User](craft4:craft\elements\User) object | uploaded by the user represented by the object.



::: code
```twig
{# Fetch assets uploaded by the user with an ID of 1 #}
{% set assets = craft.assets()
  .uploader(1)
  .all() %}
```

```php
// Fetch assets uploaded by the user with an ID of 1
$assets = \craft\elements\Asset::find()
    ->uploader(1)
    ->all();
```
:::


#### `volume`

Narrows the query results based on the volume the assets belong to.

Possible values include:

| Value | Fetches assets…
| - | -
| `'foo'` | in a volume with a handle of `foo`.
| `'not foo'` | not in a volume with a handle of `foo`.
| `['foo', 'bar']` | in a volume with a handle of `foo` or `bar`.
| `['not', 'foo', 'bar']` | not in a volume with a handle of `foo` or `bar`.
| a [craft\models\Volume](craft4:craft\models\Volume) object | in a volume represented by the object.



::: code
```twig
{# Fetch assets in the Foo volume #}
{% set assets = craft.assets()
  .volume('foo')
  .all() %}
```

```php
// Fetch assets in the Foo group
$assets = \craft\elements\Asset::find()
    ->volume('foo')
    ->all();
```
:::


#### `volumeId`

Narrows the query results based on the volumes the assets belong to, per the volumes’ IDs.

Possible values include:

| Value | Fetches assets…
| - | -
| `1` | in a volume with an ID of 1.
| `'not 1'` | not in a volume with an ID of 1.
| `[1, 2]` | in a volume with an ID of 1 or 2.
| `['not', 1, 2]` | not in a volume with an ID of 1 or 2.
| `':empty:'` | that haven’t been stored in a volume yet



::: code
```twig
{# Fetch assets in the volume with an ID of 1 #}
{% set assets = craft.assets()
  .volumeId(1)
  .all() %}
```

```php
// Fetch assets in the volume with an ID of 1
$assets = \craft\elements\Asset::find()
    ->volumeId(1)
    ->all();
```
:::


#### `width`

Narrows the query results based on the assets’ image widths.

Possible values include:

| Value | Fetches assets…
| - | -
| `100` | with a width of 100.
| `'>= 100'` | with a width of at least 100.
| `['>= 100', '<= 1000']` | with a width between 100 and 1,000.



::: code
```twig
{# Fetch XL images #}
{% set assets = craft.assets()
  .kind('image')
  .width('>= 1000')
  .all() %}
```

```php
// Fetch XL images
$assets = \craft\elements\Asset::find()
    ->kind('image')
    ->width('>= 1000')
    ->all();
```
:::


#### `with`

Causes the query to return matching assets eager-loaded with related elements.



See [Eager-Loading Elements](https://craftcms.com/docs/4.x/dev/eager-loading-elements.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch assets eager-loaded with the "Related" field’s relations #}
{% set assets = craft.assets()
  .with(['related'])
  .all() %}
```

```php
// Fetch assets eager-loaded with the "Related" field’s relations
$assets = \craft\elements\Asset::find()
    ->with(['related'])
    ->all();
```
:::


#### `withTransforms`

Causes the query to return matching assets eager-loaded with image transform indexes.

This can improve performance when displaying several image transforms at once, if the transforms
have already been generated.

Transforms can be specified as their handle or an object that contains `width` and/or `height` properties.

You can include `srcset`-style sizes (e.g. `100w` or `2x`) following a normal transform definition, for example:

::: code

```twig
[{width: 1000, height: 600}, '1.5x', '2x', '3x']
```

```php
[['width' => 1000, 'height' => 600], '1.5x', '2x', '3x']
```

:::

When a `srcset`-style size is encountered, the preceding normal transform definition will be used as a
reference when determining the resulting transform dimensions.



::: code
```twig
{# Fetch assets with the 'thumbnail' and 'hiResThumbnail' transform data preloaded #}
{% set assets = craft.assets()
  .kind('image')
  .withTransforms(['thumbnail', 'hiResThumbnail'])
  .all() %}
```

```php
// Fetch assets with the 'thumbnail' and 'hiResThumbnail' transform data preloaded
$assets = \craft\elements\Asset::find()
    ->kind('image')
    ->withTransforms(['thumbnail', 'hiResThumbnail'])
    ->all();
```
:::



<!-- END PARAMS -->
