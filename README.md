# FileManager for Yii2

**ATTENTION: Documentation in process!**

[![Build Status](https://travis-ci.org/rkit/filemanager-yii2.svg?branch=master)](https://travis-ci.org/rkit/filemanager-yii2)
[![Code Coverage](https://scrutinizer-ci.com/g/rkit/filemanager-yii2/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/rkit/filemanager-yii2/?branch=master)
[![codecov.io](http://codecov.io/github/rkit/filemanager-yii2/coverage.svg?branch=master)](http://codecov.io/github/rkit/filemanager-yii2?branch=master)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/rkit/filemanager-yii2/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/rkit/filemanager-yii2/?branch=master)

## Featuresgallery

- AJAX upload
- Support multiple file uploads
- Validations of file
- Create thumbnails (easy setup presets) on the fly with cache or after upload
- Check the owner of a file
- Storing information about a file in the database
- Ability to use any component for working with images
- Ability to change the way of storing files

## Installation

1. Installing using Composer

   ```
   composer require rkit/filemanager-yii2
   ```

2. Run migrations

   ```
   php yii migrate --migrationPath=@vendor/rkit/filemanager-yii2/src/migrations/ --interactive=0
   ```

### Documentation

[API Reference](/docs)

## Configuration

Add the following in your config, in section `components`

``` php
'fileManager' => [
    'class' => 'rkit\filemanager\FileManager',
    // directory for files inaccessible from the web
    'uploadDirProtected' => '@app/runtime',
    // directory for files available from the web
    'uploadDirUnprotected' => '@app/web',
    // path in a directory of upload
    'publicPath' => 'uploads',
    // an array of the types of owners
    // in the format of `table.attribute` => `unique value`
    'ownerTypes' => [
        'news.text' => 1,
        'news.preview' => 2,
        'news.gallery' => 3,
        'user_profile.photo' => 4,
    ]
]
```

## Usage

### Basic usage

1. **Controller**

   ``` php
   public function behaviors()
   {
       return [
           'verbs' => [
               'class' => VerbFilter::className(),
               'actions' => [
                   'preview-upload' => ['post']
               ],
           ],
       ];
   }

   public function actions()
   {
       return [
           'preview-upload' => [
               'class' => 'rkit\filemanager\actions\UploadAction',
               'modelName' => 'app\models\News',
               'attribute' => 'preview',
               // the type of the file (`image` or `file`)
               'type' => 'image',
               // the name of the file input field
               'inputName' => 'file',
           ],
       ];
   }
   ```

2. **Model**

   > The example uses [Intervention\Image](https://github.com/Intervention/image), but this is optional.  
   > You can use any library for working with files.

   ``` php
   // any component to resize/crop images
   use Intervention\Image\ImageManagerStatic as Image;
   …

   public function behaviors()
   {
       return [
           [
               'class' => 'rkit\filemanager\behaviors\FileBehavior',
               'attributes' => [
                   'preview' => [
                       // local storages
                       'storage' => 'rkit\filemanager\storages\LocalStorage',
                       // save path of the file in this table
                       'saveFilePath' => true,
                       // @see http://www.yiiframework.com/doc-2.0/guide-tutorial-core-validators.html
                       'rules' => [
                           'imageSize' => ['minWidth' => 300, 'minHeight' => 300],
                           'mimeTypes' => ['image/png', 'image/jpg', 'image/jpeg'],
                           'extensions' => ['jpg', 'jpeg', 'png'],
                           'maxSize' => 1024 * 1024 * 1, // 1 MB
                           'tooBig' => Yii::t('app', 'File size must not exceed') . ' 1Mb'
                       ],

                       // presets for the files, can be used on the fly
                       // or you can to apply after upload
                       'preset' => [
                           '200x200' => function ($realPath, $publicPath, $thumbPath) {
                               // any manipulation on the file
                               Image::make($realPath . $publicPath)
                                   ->fit(200, 200)
                                   ->save($realPath . $thumbPath, 100);
                           },
                           '220x220' => function ($realPath, $publicPath, $thumbPath) {
                               // any manipulation on the file
                               Image::make($realPath . $publicPath)
                                   ->fit(220, 220)
                                   ->save($realPath . $thumbPath, 100);
                           },
                       ],

                       // * — to apply all presets after upload
                       // or an array with the names[] of presets
                       'applyPresetAfterUpload' => '*'
                   ]
               ]
           ]
       ];
   }
   ```

3. **View**

   Any widget for ajax upload (you can use [the widget for FileApi](https://github.com/rkit/fileapi-widget-yii2))  
   **IMPORTANT**: In the value of input should be the id of a file or an array of ids.

### Gallery

1. **Controller**

   ``` php
   public function actions()
   {
       return [
           'gallery-upload' => [
               'class' => 'rkit\filemanager\actions\UploadAction',
               'modelName' => 'app\models\News',
               'attribute' => 'gallery',
               // the type of the file (`image` or `file`)
               'type' => 'image',
               // the name of the file input field
               'inputName' => 'file',
               // multiple files
               'multiple'  => true,
               // path to the template for uploaded a file
               'template'  => Yii::getAlias('@app/path/to/file')
           ]
       ]
   }
   ```

2. **Model**

   ``` php
   public function behaviors()
   {
       return [
           [
               'class' => 'rkit\filemanager\behaviors\FileBehavior',
               'attributes' => [
                   'gallery' => [
                   // local storages
                       'storage' => 'rkit\filemanager\storages\LocalStorage',
                       // multiple files
                       'multiple' => true,
                       'preset' => [
                           '80x80' => function ($realPath, $publicPath, $thumbPath) {
                               // any manipulation on the file
                               Image::make($realPath . $publicPath)
                                   ->fit(80, 80)
                                   ->save($realPath . $thumbPath, 100);
                           },
                       ],
                       'applyPresetAfterUpload' => ['80x80']
                   ]
               ]
           ]
       ];
   }
   ```

3. **Template** for uploaded a file

   ``` php
   <li>
     <a href="<?= $file->getStorage()->path()?>" target="_blank">
       <img src="<?= $model->thumb('gallery', '80x80', $file->getStorage()->path())?>">
     </a>
     <?= Html::textInput(Html::getInputName($model, $attribute) . '[' . $file->id .']', $file->title, [
         'class' => 'form-control',
     ])?>
   </li>

   ```

### Save a file in a storage immediately after upload

For example, this could be a need for wysiwyg editor,  
when you need to immediately save the file after upload and assign the owner.

``` php
public function actions()
{
    return [
        'text-upload' => [
            'class' => 'rkit\filemanager\actions\UploadAction',
            'modelName' => 'app\models\News',
            'attribute' => 'text',
            'type' => 'image',
            'inputName' => 'file',
            'temporary' => false,
            'ownerId' => 0,
        ]
    ]
}
```
