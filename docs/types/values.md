---
sidebar_position: 2
---

# Values

All JSON primitive datatypes are supported in an Automerge document.

*Remember, never modify `currentDoc` directly, only ever change `doc`!*

```js
newDoc = Automerge.change(currentDoc, doc => {
  doc.property = 'value' // assigns a string value to a property
  doc['property'] = 'value' // equivalent to the previous line
  delete doc['property'] // removes a property

  doc.stringValue = 'value'
  doc.numberValue = 1
  doc.boolValue = true
  doc.nullValue = null
  doc.nestedObject = {} // creates a nested object
  doc.nestedObject.property = 'value'
  // you can also assign an object that already has some properties
  doc.otherObject = { key: 'value', number: 42 }
})
```