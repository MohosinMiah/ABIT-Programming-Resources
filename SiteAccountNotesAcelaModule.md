# Reference Ticket
https://jupiterplatform.com/Tickets/edit.php?id=57097&siteid=1108

# Acela Module Creation Guide

This document outlines the process for creating a new module in the Acela framework. Follow the steps below to ensure proper structure and consistency across all modules.

## Step 1: Create a Migration File

1. **File Naming Convention**:

   - The migration file should be located in the `csm-stafftools/Migration` folder.
   - The table name should follow `camelCase` format and be **plural**.
   - Column names should:
     - Be in `camelCase` format.
     - Be **singular**.
     - Include the singular form of the table name as the first part.

   **Example**:
   For a table named `siteNotes`, the column storing the title should be named `siteNoteTitle`.

   **SQL Code Example**:

   ```sql
   CREATE TABLE siteNotes (
       siteNoteId INT AUTO_INCREMENT PRIMARY KEY,
       siteNoteTitle VARCHAR(255) NOT NULL,
       siteNoteContent TEXT NOT NULL,
       createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

   **PHP Migration Example**:

   ```php
   <?php
   /**
    *  Migration for Site Notes.
    */

   namespace Csm\Acela\StaffTools\Migrations;

   use Acela\Core;

   class SiteNotes extends Core\MigrationAbstract
   {
       public function listVersions()
       {
           return [
               '20231211_120000', // Initial migration for siteNotes table.
           ];
       }

       public function listSchemas()
       {
           return [
               'siteNotes',
           ];
       }

       public function schemaSiteNotes($schemaTable)
       {
           $schemaTable
               ->bigInt('siteNoteId')->primaryKey()->autoIncrement()
               ->varText('siteNoteTitle', 255)->notNullable()->index()
               ->varText('siteNoteContent', 65535)->notNullable()
               ->dateTime('createdAt')->notNullable()->index();
       }
   }
   ```

2. **Command to Create Database**:

   - Navigate to the Acela directory in your command line.
   - Run the following command:
     ```bash
     ./acli migrate upgrade
     ```

## Step 2: Create a Controller

1. **Location**:

   - Place the controller in the `Controller` folder. It could be under legacyDatabase or stafftools

2. **Field Names in the Controller**:

   - Use the last part of the `camelCase` column names, excluding the singular form of the table name.
   - Ensure all field names are in lowercase.

   **Example**:
   For a column `siteNoteTitle`, the field name in the controller would be `title`.

   **Code Example**:

   ```php
    <?php
    namespace Csm\Acela\StaffTools\Controllers;

    use Acela\Core;
    use Acela\Admin\Controllers\Dashboard;
    use Csm\AcelaSiteSwitch\SiteSwitch;

    /**
     *  Controller for API Keys
     */
    class ApiKeys extends Core\ControllerAbstract
    {
        /**
         *  Controller to display a list.
         *  
         *  @param Core\Request $request The page request.
         */
        protected function listItems( Core\Request $request )
        {
            /**
             * Check permissions.
             */
            Core\Permission::requirePermission( 'admin' );

            /**
             * Create a datatable.
             */
            $table = Core\Table::create();
            $table->name = 'apiKeys';
            $table->title = 'API Keys';

            /**
             * Query to get API keys.
             */
            $table->query =
                Core\Query::create()
                    ->table( 'apiKeys', 'ak' )
                    ->where( [ 'ak', 'apiKeyDeleted' ], '=', false )
                    ->where( [ 'ak', 'siteId' ], '=', SiteSwitch::getSite() )
            ;

            /**
             * Specify filters.
             */ 
            $form = Core\Form::create(); // Create a new form to hold the filters.
            $form->name = 'apiKeys';
            $form->title = 'API Keys';

            /**
             * Specify columns.
             */
            $column = $table->column();
            $column->name = 'title';
            $column->title = 'Title';
            $column->value = function ( $row ) { return $row['apiKeyTitle']; };
            $column->sortGetParameter = 'title';
            $column->sortable =
                function ( $dataTable, $sortType )
                {
                    $dataTable->query->sort( 'ak', 'apiKeyTitle', $sortType );
                }
            ; 

            $column = $table->column();
            $column->name = 'uuid';
            $column->title = 'UUID';
            $column->value = function ( $row ) { return $row['apiKeyUuid']; };
            $column->sortGetParameter = 'uuid';
            $column->sortable =
                function ( $dataTable, $sortType )
                {
                    $dataTable->query->sort( 'ak', 'apiKeyUuid', $sortType );
                }
            ; 


            /**
             * Add buttons to the table.
             */
            $button = $table->button();
            $button->name = 'add';
            $button->title = 'New API Key';
            $button->type = 'link';
            $button->link = Core\Link::build( $request->url . '/add', [], true );

            /**
             * Specify the view to be used.
             */
            Core\Template::setView( 'ApiKeyList' );
            $params = [
                'dataTableFilters' => $form->run(),
                'dataTable' => $table->run(),
                'layout' => [
                    'title' => Dashboard::buildBreadcrumbs( 'agency/api/keys' ),
                ],
            ];
            Core\Template::addParameters( $params );
            Core\Template::run();
        }

        /**
         *  Controller to add a new record.
         *  
         *  @param Core\Request $request The page request.
         *  @param mixed $ajax The AJAX mode to use. False if not using AJAX.
         */
        protected function addItem( Core\Request $request )
        {
            return $this->editItem( $request, 0, 'add' ); // Pass the call through to the edit method, with the 'add' mode specified.
        }
        
        /**
         *  Controller to edit a record.
         *  
         *  @param Core\Request $request The page request.
         *  @param int $id The id of the record you wish to edit.
         *  @param string $mode The mode (add, edit, delete).
         *  @param mixed $ajax The AJAX mode to use. False if not using AJAX.
         */
        protected function editItem( Core\Request $request, $id, $mode = 'edit' )
        {
            /**
             *  Check permissions.
             */
            Core\Permission::requirePermission( 'admin' );
            
            /**
             *  Set up the form
             */
            $form = Core\Form::create(); // Create a new form.
            
            /**
             *  Get or create a record.
             */
            if( $mode === 'add' )
            {
                $record = Core\Model::getInstance( 'ApiKey' )->create(); // Create a new record.
            }
            else
            {
                $record = Core\Model::getInstance( 'ApiKey' )->getFirst( [ 'uuid' => $id, 'siteId' => SiteSwitch::getSite(), 'deleted' => false ] ); // Get the appropriate record.
            }

            if( !$record ) // If this is not a valid record...
            {
                Core\Response::redirect404();
            }

            $form->object = $record; // Attach the record to the form.

            /**
             * Set up name and title.
             */
            $form->name = 'formApiKey' . ucfirst( $mode );
            $form->title = ucfirst( $mode ) . ' API Key';

            /**
             * Set read-only state.
             */
            if( in_array( $mode, [ 'view','delete' ], true ) )
            {
                $form->readOnly = true; // Make the form read-only.
            }

            /**
             * Add form fields.
             */
            $field = $form->field();
            $field->name  = 'title';
            $field->title = 'Title';
            $field->type  = 'text';
            $field->tip   = 'A short description for this API key.';
            
            /**
             * Add buttons.
             */
            $callback = // Create a common callback function.
                function () use ( $form, $mode )
                {
                    if( $mode == 'add' )
                    {
                        $form->object->siteId = SiteSwitch::getSite();
                        $form->object->uuid = Core\uuidSecure();
                        $form->object->key = bin2hex( random_bytes( 128 ) );

                        Core\Alert::warning( 'Your new API key is ' . $form->object->key . ' . Please save this for your reference, as this key will never be displayed again.', true );
                    }
                }
            ;
            if( $mode === 'delete' ) // If we're on a delete page...
            {
                $button = $form->button();
                $button->name = 'delete';
                $button->title = 'Delete';
                $button->type = 'delete';
            }
            elseif( $mode === 'edit' ) // Normal edit mode...
            {
                $button = $form->button();
                $button->name = 'save';
                $button->title = 'Save';
                $button->type = 'save';
                $button->callback = $callback;
            }
            elseif( $mode === 'add' ) // If we're adding a new entry...
            {
                $button = $form->button();
                $button->name = 'add';
                $button->title = 'Add';
                $button->type = 'save';
                $button->callback = $callback;
            }
            
            /**
             *  Build the form.
             */
            $form->run();
            
            /**
             * Set up the view.
             */
            Core\Template::setView( 'ApiKeyEdit' );
            $params = [
                'form' => $form,
                'layout' => [
                    'title' => array_merge(
                        Dashboard::buildBreadcrumbs( 'agency/api/keys' ),
                        [
                            [ ucfirst( $mode ).' Key', Core\Link::build() ],
                        ]
                    ),
                ],
            ];
            Core\Template::addParameters( $params );
            Core\Template::run();
        }
    }

   ```

3. **Logic and Data Handling**:

   - Add appropriate logic to process data within the controller.
   - Retrieve necessary data from the database and pass it to the view.

## Step 3: Add Routes to bootstrap.php
Location:
Modify the bootstrap.php file to include new routes for the module.
Code Example:
```
/**
 * site notes
 */
$basePath = 'siteNotes';
$controller = Controllers\SiteNotes::class;
Core\Route::create('any', $basePath . '/add',                         [ $controller, 'addItem' ],   'admin');
Core\Route::create('any', $basePath . '/([0-9]+)/(view|edit|delete)', [ $controller, 'editItem' ],  'admin');
Core\Route::create('any', $basePath . '(/?)',                         [ $controller, 'listItems' ], 'admin');

```
## Step 4: Create Related View Files

1. **Location**:

   - Place the view files in the `View` folder.

2. **Logic Placement**:

   - Ensure the view files only contain view-specific logic.
   - Data sent from the controller should be appropriately utilized to generate the desired output.

   **Code Example**:

   ```php
   {@extends:Main}

   <?php
   /**
   * Display the search field with a special class.
   */
   $dataTableFilters->extraClasses[] = 'formFilters';

   /**
    * Add links to certain columns.
    */
   foreach( $dataTable->data  as $rowNum => &$row )
   {
       $links = [
           'title' => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/edit', [] ),
           'uuid'  => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/edit', [] ),
           'key'   => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/edit', [] ),
       ];
       $row['actions'] = [
           [
               'title' => 'View',
               'name'  => 'view',
               'link'  => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/view' ),
           ],
           [
               'title' => 'Edit',
               'name'  => 'edit',
               'link'  => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/edit' ),
           ],
           [
               'title' => 'Delete',
               'name'  => 'delete',
               'link'  => Core\Link::build( 'agency/api/keys/' . $row['uuid'] . '/delete' ),
           ],
       ];
       foreach( $links as $key => $link )
       {
           $row[ $key ] = '<a href="' . $link . '">' . $row[ $key ] . '</a>';
       }
   }

   /**
   * Do not show title of the main table.
   */
   $dataTable->showTitle = false;
   ?>
   {@view:Table/Table}
   ```

## Summary

1. **Migration**:

   - Create a migration file with table and column names following the specified naming conventions.
   - Store the migration file in `csm-stafftools/Migration`.

2. **Controller**:

   - Create a controller in the `Controller` folder.
   - Define field names using only the relevant part of the column name, excluding the table name prefix.
   - Implement logic to retrieve and pass data to the view.
   - Store the migration file in `csm-stafftools/Controller` or `csm-legacydatabase/Controller`.


3. **View**:

   - Create view files in the `View` folder.
   - Ensure view files only handle view-specific logic.

4. **Database Migration Command**:

   - Run the migration using:
     ./acli migrate upgrade

By following this guide, you can create and maintain modules in Acela efficiently and consistently.


