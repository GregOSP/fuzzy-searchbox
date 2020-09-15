# fuzzy-searchbox

This component creates an accessible screen-reader-friendly autocomplete combobox with fuzzy searching.

Dev Dependencies:
  - Fuse.js
  - Lodash
  - node-sass
  - Vue

Notes:
  - This component expects a list of objects as input. ('items')
  - A future update should allow items to be a list of strings as well in cases with simpler dropdown options
  - An array of keys needs to be passed in so the component knows what keys to filter on. ('itemKeys')
  - There are two options for specifying the labels for the items
    1) 'itemLabels' can be passed in as an array of keys (just like 'itemKeys'). The label will be the concatenation of the key values. The delimiter can be passed in as 'labelDelimiter', otherwise the default is a single space.
    2) 'createLabelFunction' can be passed in as a function that accepts an item and returns a string label.
  - 'searchByLabel' is an optional flag to use the label string for search instead of using the itemKeys, useful when used with 'createLabelFunction'
    - 'searchByLabelRuleFunction' is an optional parameter that can be passed in as a function that modifies the underlying search string
  - 'preSelectedItem' should be set if there are events that pass an item selection into the FuzzySearchbox module from outside
  - This compenent requires an aria-label to be set
  - Optionally set 'placeholder' as text when the input box is empty
  - Optional: set 'max-items-displayed' to limit the number of search results
  - Optional: set 'show-default-results' to false to prevent the search results box from opening when the input is empty
  - Optional: set 'persist-search' to false reset the search string after the user makes a selection, showing the default results next time they click

#Github
```
https://github.com/GregOSP/fuzzy-searchbox
```

#Installation
```
$ npm i fuzzy-searchbox
```

#Sample usage

```javascript
import FuzzySearchbox from fuzzy-searchbox
```

```html
<FuzzySearchbox
  id="my-select"
  aria-label="select an option"
  placeholder="Type here to search for an option"
  v-model="selectedOption"
  :preSelectedItem="defaultItem"
  @change="optionChanged"
  :items="selectList"
  :item-keys="['name', 'type', 'category']"
  :searchByLabel = "true"
  :searchByLabelRuleFunction = "labelModificationFunction"
  :create-label-function="displayStringForLabel"
  :max-items-displayed = 10
  :show-default-results = "true">
</FuzzySearchbox>
```