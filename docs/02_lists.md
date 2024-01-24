# Lists

Creating observable lists in **Impact** is straight forward. It is just a signal with an array.

```ts
function AppContext() {
    const list = signal([])

    return {
        get list() {
            return list.value
        }
    }
}
```

Since signal values are considered immutable, like in React, you update that list by:

```ts
function AppContext() {
    const list = signal([])

    return {
        get list() {
            return list.value
        },
        addToList(item) {
            list.value = [item, ...list.value]
        }
    }
}
```

But sometimes lists needs to update complex objects within that list. In this scenario it can be a good idea to use something like [immer](https://immerjs.github.io/immer/):


```ts
import { produce } from 'immer'

function AppContext() {
    const list = signal([])

    return {
        get list() {
            return list.value
        },
        addToList(item) {
            produce(list.value, (draft) => {
                draft.unshift(item)
            })
        },
        changeTitle(index, newTitle) {
            produce(list.value, (draft) => {
                draft[index].title = newTitle
            })
        }
    }
}
```

Most lists can be managed this way, but you might have much bigger lists. Lists with sorting, filters and other  states. You want to ensure maximum performance so that the component managing the list does not reconcile when items in the list updates and you want full control of what items are shown in the list at any moment.

```tsx
// In this example you receive the list of initial items as a prop
// to the context
function AppContext({ initialItems }) {
    const { api } = useGlobalContext()

    // We create a signal for managing what to display in the list. It
    // only contains ids to items
    const listById = signal([])
    
    // We create a dictionary to reference items
    const items = initialItems.reduce((acc, item) => {
        // We turn every item into a signal
        acc[item.id] = signal(item)

        return acc
    })

    // We subscribe to updates
    cleanup(api.subscribeItemsUpdate((updatedItem) => {
        items[updatedItem.id].value = updatedItem
    }))


    return {
        get listById() {
            return listById.value
        },
        getItemById(id) {
            return items[id].value
        }
    }
}

const useAppContext = context(AppContext)

function ListComponent() {
  const { listById } = useAppContext()

  return <ul>{listById.map((id) => <ItemComponent key={id} id={id} />)}</ul>
}

// You memoize so that when the list changes you only reconcile if the ID
// of the item actually changes
const ItemComponent = memo(function ItemComponent({ id }) {
  const { getItemById } = useAppContext()
  // You consume the specific item directly from the context
  const item = getItemById(id)

})
```

Now you are free to choose how to produce the list to display. By iterating the `items` record you can sort, filter or even add items to the list based on certain interactions. Each item is memoized on its id and the item is only consumed from within the `ItemComponent`, isolating updates.

If each item in and of itself is complex you can consider creating a context around each item in the list, but at this point it would be surprising if each item in the list could not depend on normal props passing and memoization of nested components to optimise reconciliation.