# Relay

## Goals

* Colocate data requirements with components that use them
* Separate data fetching from presentation
* Enable efficient re-renders through fragment masking
* Test GraphQL components without network mocks

## Fragment colocation

Each component declares its own data needs via fragments. The parent query
spreads the fragment without knowing what fields are inside.

```typescript
// TripCard.tsx — declares its own data requirements
import { graphql, useFragment } from 'react-relay'
import type { TripCard_trip$key } from './__generated__/TripCard_trip.graphql'

const tripFragment = graphql`
  fragment TripCard_trip on trip {
    id
    goals
    created_at
  }
`

interface Props {
  trip: TripCard_trip$key
  onPress?: (tripId: string) => void
}

export function TripCard({ trip: tripRef, onPress }: Props) {
  const trip = useFragment(tripFragment, tripRef)
  return <Card onPress={() => onPress?.(trip.id)}>{trip.goals?.[0]}</Card>
}
```

The parent spreads the fragment:

```typescript
// TripListScreen.tsx — spreads child fragment
const tripListQuery = graphql`
  query TripListScreenQuery {
    tripCollection {
      edges {
        node {
          nodeId
          ...TripCard_trip
        }
      }
    }
  }
`
```

Benefits:

* Components are self-contained — move or delete without hunting for queries
* Parent does not need to know child's data requirements
* Relay masks fragment data, preventing accidental coupling

## Container/Display pattern with Relay

### Container component

Handles data fetching and business logic. Query is colocated with the container.

```typescript
// TripListScreenContent.tsx
export function TripListScreenContent() {
  const router = useRouter()
  const data = useLazyLoadQuery<TripListScreenQuery>(tripListQuery, {})

  const handleViewTrip = (tripId: string) => {
    router.push(`/trips/${tripId}`)
  }

  return (
    <YStack>
      {data.tripCollection?.edges?.map((edge) => (
        <TripCard
          key={edge.node.nodeId}
          trip={edge.node}
          onPress={handleViewTrip}
        />
      ))}
    </YStack>
  )
}
```

### Display component

Pure presentation via fragment. Does not know about routing or business logic.

```typescript
// TripCard.tsx
export function TripCard({ trip: tripRef, onPress }: Props) {
  const trip = useFragment(tripFragment, tripRef)
  return <Card onPress={() => onPress?.(trip.id)}>{trip.goals[0]}</Card>
}
```

## Suspense boundaries

Wrap containers in Suspense to handle loading state:

```typescript
export function TripListScreen() {
  return (
    <Suspense fallback={<Spinner size="large" />}>
      <TripListScreenContent />
    </Suspense>
  )
}
```

Place boundaries close to the component that fetches data. This allows
independent loading states for different parts of the UI.

## Query naming

Query names must match the filename. The Relay compiler enforces this.

```typescript
// trip-list-screen.tsx
const tripListScreenQuery = graphql`
  query tripListScreenQuery { ... }  // matches filename
`
```

Fragment names follow the pattern `ComponentName_fieldName`:

```typescript
// TripCard.tsx
fragment TripCard_trip on trip { ... }
```

## Mutations

```typescript
import { graphql, useMutation } from 'react-relay'

const createTripMutation = graphql`
  mutation CreateTripMutation($input: tripInsertInput!) {
    insertIntotripCollection(objects: [$input]) {
      records {
        nodeId
        id
        goals
      }
    }
  }
`

function useCreateTrip() {
  const [commit, isInFlight] = useMutation(createTripMutation)

  const createTrip = (goals: string[]) => {
    commit({
      variables: { input: { goals } },
      onCompleted: (response) => { /* handle success */ },
      onError: (error) => { /* handle error */ },
    })
  }

  return { createTrip, isCreating: isInFlight }
}
```

Include `nodeId` in mutation responses to enable automatic cache updates.

## Testing Relay components

Use provider-based dependency injection instead of `jest.mock()`:

```typescript
import {
  TestRelayProvider,
  MockPayloadGenerator,
  createTestRelayEnvironment,
} from '../test-utils/relay'

test('renders trip list', async () => {
  const environment = createTestRelayEnvironment()

  render(
    <TestRelayProvider environment={environment}>
      <TripListScreen />
    </TestRelayProvider>
  )

  // Resolve query with mock data
  await act(() => {
    environment.mock.resolveMostRecentOperation((operation) =>
      MockPayloadGenerator.generate(operation, {
        trip: () => ({
          nodeId: 'test-node-id',
          id: 'trip-123',
          goals: ['Visit Paris'],
        }),
      })
    )
  })

  expect(screen.getByText('Visit Paris')).toBeTruthy()
})
```

This approach:

* Avoids `jest.mock()` which is a code smell
* Uses the same provider pattern as production
* Allows fine-grained control over mock responses

## Notes

* Query names must match filename (Relay compiler requirement)
* See [component-structure](./component-structure.md) for general container/display
  patterns
* See [testing](./testing.md) for dependency injection principles
