# Title

```js
const Nuxt = require('nuxt')
const nuxt = new Nuxt()

//Lorem ipsum dolor sit amet, consectetur adipiscing elit. Curabitur congue elit vel nisl euismod, in fermentum tellus convallis. Nam quis ex id augue ornare facilisis in eget mauris. In porta non odio vitae volutpat. Integer hendrerit, ligula at convallis pellentesque, nunc mauris varius nisi, vitae rutrum augue nisl a nunc. Praesent at ornare neque, ac tincidunt nisl. 

nuxt.renderAndGetWindow('http://localhost:3000')
.then((window) => {
  // Display the head <title>
  console.log(window.document.title)
})
```

- #### `runInNewContext`

  - 2.3.0+
  - only used in `createBundleRenderer`
  - Expects: `boolean | 'once'` (`'once'` only supported in 2.3.1+)
