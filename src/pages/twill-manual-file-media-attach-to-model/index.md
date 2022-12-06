---
title: Twill CMS file/media manual attach to the model
date: '2022-12-07'
spoiler: How to manually append file/media to model without library
---

This topic is frequent on the Twill official [Discord](https://discord.link/twill) server, so here is a few ways to achieve this.

In all cases, you will need to create a `File` or `Media` Twill model manually.

So let's start with that.

```php
// File example
// $uuid - unique id, you can get it via Str::uuid();
// $filename - uploaded file name
// $size - byte filesize

$file = new \A17\Twill\Models\File([
    'uuid' => $uuid . '/' . $filename,
    'filename' => $filename,
    'size' => $size,
    'created_at' => Carbon::now()->format('Y-m-d H:i:s'),
    'updated_at' => Carbon::now()->format('Y-m-d H:i:s')
]);
```

```php
// Media example
// $uuid - unique id, you can get it via Str::uuid();
// $filename - uploaded media name

$media = new \A17\Twill\Models\Media([
    'uuid' => $uuid . '/' . $filename,
    'alt_text' => '',
    'width' => '',
    'height' => '',
    'caption' => '',
    'filename' => $filename,
    'created_at' => Carbon::now()->format('Y-m-d H:i:s'),
    'updated_at' => Carbon::now()->format('Y-m-d H:i:s')
]);
```

Twill uses polymorphic relation to create a relation between your model and the selected file/media.

Firstly you will need to get a model instance of your own model.

```php
// For files
$model = YourModel::find(1);

$model->files()->save($file, 
                            [
                              'locale' => locale(), 
                              'role' => 'your_file_input_name_on_module_form'
                            ]);
```

```php
// For media
$model = YourModel::find(1);

$model->medias()->save($media, 
                            [
                              'locale' => locale(), 
                              'role' => 'your_media_input_name_on_module_form',
                              'crop_x' => '',
                              'crop_y' => '',
                              'crop_w' => '',
                              'crop_h' => '',
                              'crop' => 'default',
                              'lqip_data' => null,
                              'ratio' => '',
                              'metadatas' => '{}'
                            ]);
```

You can always back up to the `DB` facade to achieve this same thing.

```php
// This example is for files only, the same logic is for the media
$fileId = DB::table('files')->insertGetId([
    'uuid' => $uuid . '/' . $filename,
    'filename' => $filename,
    'size' => $size,
    'created_at' => Carbon::now()->format('Y-m-d H:i:s'),
    'updated_at' => Carbon::now()->format('Y-m-d H:i:s')
]);

//Insert to fileables table
DB::table('fileables')->insert([
    'fileable_id' => $id,
    'fileable_type' => 'App\Models\YourModel',
    'file_id' => $fileId,
    'role' => 'your_file_input_name_on_module_form',
    'locale' => locale(),
    'created_at' => Carbon::now()->format('Y-m-d H:i:s'),
    'updated_at' => Carbon::now()->format('Y-m-d H:i:s')
]);
```