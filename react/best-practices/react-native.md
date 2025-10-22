# React Native / Expo Patterns

## Container/Display with Multiple Data Sources

Display components accept data as props. Containers fetch from different sources:

### Display Component

```typescript
interface PlanDetailsDisplayProps {
  plan: PlanResponse | null
  loading?: boolean
  error?: Error | null
}

export function PlanDetailsDisplay({ plan, loading, error }: PlanDetailsDisplayProps) {
  const [expandedDays, setExpandedDays] = useState<Set<number>>(new Set([0]))

  if (loading) return <YStack accessibilityLabel="Loading"><Spinner /></YStack>
  if (error) return <YStack accessibilityRole="alert">{error.message}</YStack>
  if (!plan) return <YStack>Not found</YStack>

  return <YStack>{/* render */}</YStack>
}
```

### Container 1: React Query Cache

```typescript
export function QueriedPlanDetails() {
  const { planIndex } = useLocalSearchParams<{ planIndex: string }>()
  const queryClient = useQueryClient()
  const plans = queryClient.getQueryData<PlanResponse[]>(['plans'])
  const plan = plans?.[parseInt(planIndex || '0', 10)] ?? null

  return <PlanDetailsDisplay plan={plan} />
}
```

### Container 2: AsyncStorage with Cleanup

```typescript
export function SavedPlanDetails() {
  const { id } = useLocalSearchParams<{ id: string }>()
  const storage = useStorageContext()
  const [data, setData] = useState<Data | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    let cancelled = false  // Cleanup pattern
    storage.get(id)
      .then(d => !cancelled && setData(d))
      .catch(e => !cancelled && setError(e))
      .finally(() => !cancelled && setLoading(false))
    return () => { cancelled = true }
  }, [id, storage])

  return <PlanDetailsDisplay plan={data?.plan} loading={loading} error={error} />
}
```

## Accessibility Essentials

```typescript
// Loading with announcement
<YStack accessibilityLabel="Loading content">
  <Spinner />
</YStack>

// Error with live region
<YStack accessibilityRole="alert" accessibilityLiveRegion="polite">
  <Paragraph>{error.message}</Paragraph>
</YStack>

// Interactive elements
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Expand details"
  accessibilityHint="Shows additional information"
  onPress={onPress}
>
  <Icon />
</Pressable>
```

## Testing Without jest.mock

Use providers instead of mocking:

```typescript
// Test display component (no mocking needed)
test('renders loading state', () => {
  render(<Display plan={null} loading={true} />)
  expect(screen.getByText('Loading')).toBeTruthy()
})

// Test container with provider injection
test('loads from storage', async () => {
  const mockStorage = { get: jest.fn().mockResolvedValue(mockData) }
  const MockRouter = ({ children }) => {
    React.useEffect(() => {
      // Set route params in test
      global.mockRouteParams = { id: '123' }
    }, [])
    return children
  }

  render(
    <StorageProvider value={mockStorage}>
      <MockRouter>
        <SavedPlanDetails />
      </MockRouter>
    </StorageProvider>
  )

  await waitFor(() => expect(screen.getByText(mockData.title)).toBeTruthy())
})
```
