---
title: "Combining React Router and Shopify App Bridge Navigation"
date: 2019-02-28
tags: shopify javascript
---

In this article we will show you a quick way to navigate correctly inside your react embedded app using shopify app-bridge and polaris libraries.

## Linking links

The first thing to do is to link the react router `Link`  component with shopify polaris `Link`. To do so, we pass an Adapter to our `AppProvider` component. Shopify anticipated this issue so you only have to pass the Adapter to the `AppProvider` component under the attribute `linkComponent`. Here is an example :

```jsx
import { Link } from 'react-router-dom'
import { AppProvider } from '@shopify/polaris'

function AdapterLink ({ url, ...rest }) {
  return <Link to={url} {...rest} />
}

ReactDOM.render(
  <AppProvider
    apiKey={props.apiKey}
    shopOrigin={props.shopOrigin}
    linkComponent={AdapterLink}
  >
  ...
  </AppProvider>
)
```

## A history of wrap

[Here is how to use the app bridge history object to navigate inside your app.](https://help.shopify.com/en/api/embedded-apps/app-bridge/actions/navigation/history)

Now what we want is to provide to the history the url given to our `Route` component.

### Accessing the history object

The doc does not tell us how to access the history from inside a react component. But it is actually possible to access it via [React contexts](https://reactjs.org/docs/context.html) if your component is between two `AppProvider` components by adding this :

```jsx
static contextTypes = {
  polaris: PropTypes.object
}
```
now you can call the `appBridge` inside your class, like for example in the constuctor :
```jsx
constructor(props,context) {
  super(props, context)
  const app = this.context.polaris.appBridge
}
```
### Wrapper

To combine both things we have to push the url of the `Route` inside the polaris history. So what we did is creating a wrapped component that does that every time he receives a new url :

```jsx
import { History } from '@shopify/app-bridge/actions'
import { Route } from 'react-router-dom'

function withShopifyEmbeddedAppPushState (WrappedComponent) {
  return class extends React.Component {
    static contextTypes = {
      polaris: PropTypes.object
    };

    componentWillReceiveProps (nextProps) {
      const app = this.context.polaris.appBridge
      const history = History.create(app)
      history.dispatch(History.Action.PUSH, nextProps.computedMatch.url)
    }

    render () {
      return <WrappedComponent {...this.props} />
    }
  }
}

const ShopifyEmbeddedAppRoute = withShopifyEmbeddedAppPushState(Route)
```


## Et voilà !

Here is what your rendering should like now :

```jsx
ReactDOM.render(
   <AppProvider
     apiKey={props.apiKey}
     shopOrigin={props.shopOrigin}
     linkComponent={AdapterLink}
   >
     <BrowserRouter>
       <div>
         <Switch>
           <ShopifyEmbeddedAppRoute
             path='/products'
             render={p => <ProductsContainer {...p} {...props} />}
           />
         </Switch>
       </div>
     </BrowserRouter>
   </AppProvider>
)
```

Now if you place links inside your react app :

```jsx
import { Link } from '@shopify/polaris'
<Link url='/products'>Subscription Products</Link>
```

it will call our wrapped component that will push the correct url inside shopify history !
You are now in the right page with the correct url 🕺
