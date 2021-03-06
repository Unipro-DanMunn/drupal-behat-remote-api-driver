## Handling Custom Field Types / Structures
[Back to the navigation](https://github.com/kirschbaum/drupal-behat-remote-api-driver#documentation)

### Tabular Data

Although this won't be required on the typical Drupal site, this library offers support for custom tabular data. For example, you may have one or more custom Drupal fields that aren't formatted in the traditional way and you'll need to make some adjustments to get things working and storing properly on the remote site.

First, we want the tester to be able to provide us with this custom data per field. This is already built into the library.

Example 1:

```yml
Given the fieldset "custom_field_1" with the options:
      | OPTION          | VALUE     |
      | autoplay        | 1         |
      | autoload        | 1         |
      | another         | null      |
      | even_more       | null      |
```

Example 2:

```yml
Given the fieldset "field_1" with the tabs:
      | TAB TITLE             | TABLE BODY                             | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |

Given the fieldset "field_2" with the tabs:
      | TAB TITLE             | TABLE BODY                             | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |

Given the fieldset "field_3" with the tabs:
      | TAB TITLE             | TABLE BODY                          | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |

Given the fieldset "field_4" with the tabs:
      | TAB TITLE             | TABLE BODY                             | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    | 
      | A Great Title!        | <p>A bunch of awesome body text</p>    |

```

This custom data is stored in a $custom_tables array that is keyed by the field name provided. The data itself is in a Behat TableNode object, so it's quite pleasant to work with. It's made available to you using a custom formatter.

### Adding A Custom Formatter

**Side Note:** If you think whatever problem you are solving with this formatter should be part of the core library, just open up a ticket and we can consider adding it.

Create a new class in your project (you can store it wherever you want, just make sure it gets autoloaded). The class should implement Kirschbaum\DrupalBehatRemoteAPIDriver\CustomFormatterInterface:

```php
<?php namespace DrupalRemoteAPI;

use Kirschbaum\DrupalBehatRemoteAPIDriver\CustomFormatterInterface;

class FieldFormatter implements CustomFormatterInterface {

    /**
     * Process custom Drupal data formats
     *
     * @param $info The field info for the current field in question.
     * @param $new_entity The node object being built up and formatted prior to the request.
     * @param $param The field machine name.
     * @param $column The first defined column for the field in question.
     * @param $value The value for the field in question.
     * @param $custom_data_tables Array of table objects added by custom steps.
     * @return Response The updated node object.
     * @throws \Exception
     */
    public function process($info, $new_entity, $param, $column, $value, $custom_data_tables)
    {

        // Do any necessary formatting here. See example below.

        return $new_entity;

    }

}

```

In order for the library to know to include your custom formatter we need to add the "custom_formatter_class" to the behat.yml file:

```yml
      Kirschbaum\DrupalBehatRemoteAPIDriver\DrupalRemoteExtension:
        blackbox: ~
        api_driver: 'drupal_remote_api'
        drupal_remote_api:
          login_username: 'drupal_username'
          login_password: 'drupal_password'
          custom_formatter_class: '\DrupalRemoteAPI\FieldFormatter'
        default_driver: 'drupal_remote_api'

```

### Example Use Case

Let's use the first example above. Here is what the test might look like:

```yml
Given the fieldset "custom_field_1" with the options:
      | OPTION          | VALUE     |
      | autoplay        | 1         |
      | autoload        | 1         |
      | another         | null      |
      | even_more       | null      |

Then I am viewing an "Article" node:
      | title                | A great Title                   |
      | my_subtitle          | A great subtitle                |
      | my_event_description | A great event description here. |
      | custom_field_1       | Fieldset                        |

```

The title, subtitle, and event description would work fine out of the box. But custom_field_1 and all of it's columns need a formatter.

```php
<?php namespace DrupalRemoteAPI;

use Kirschbaum\DrupalBehatRemoteAPIDriver\CustomFormatterInterface;

class FieldFormatter implements CustomFormatterInterface {

    /**
     * Process custom Drupal data formats
     *
     * @param $info The field info for the current field in question.
     * @param $new_entity The node object being built up and formatted prior to the request.
     * @param $param The field machine name.
     * @param $column The first defined column for the field in question.
     * @param $value The value for the field in question.
     * @param $custom_data_tables Array of table objects added by custom steps.
     * @return Response The updated node object.
     * @throws \Exception
     */
    public function process($info, $new_entity, $param, $column, $value, $custom_data_tables)
    {
    
        // Special handling for our custom columns.
        // We're only going to do something if the field belongs to the my_custom_field module
        if ('my_custom_field' === $info['module']) {

            // We only want to take action if custom_data_tables were provided by the tester in the form of custom steps.
            if(isset($custom_data_tables[$param]))
            {
                // ignore any placeholder value that was provided (e.g. the "Fieldset" entered above).
                $new_entity->{$param} = []; 
                // Get the appropriate table based on the field in question.
                $table = $custom_data_tables[$param];
                // For each custom column that was provided, we format it as RestWS requires.
                foreach ($table->getHash() as $hash) {
                  $new_entity->{$param}[$hash['OPTION']] = $hash['VALUE'];
                }
            } else {
                // If the field exists but the custom data wasn't provided, we let folks know how to fix it.
                throw new \Exception(sprintf('Option data not set for field "%s". There is a custom step to set option data for this field.', $param));
            }
        }
        
        return $new_entity;

    }

}
```
