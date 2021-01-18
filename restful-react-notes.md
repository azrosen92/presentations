---
theme: base-theme
style: |
  section {
    background-color: #e6f6f2;
  }
---

# Using `restful-react`

----------------

New npm script:
```
"tms:generate:client": "npx restful-react import --file ../tms/swagger/v2/swagger.yaml --output src/apps/tms/types/apiTypes.tsx"
```

-------------------

# Generated client code

This is a relatively simple endpoint with a very simple
schema, but the whole client was automatically generated in about 80 LOC.

```
# rake routes | grep tracking
  tracking_shipment POST   /:version/shipments/:id/tracking(.:format)  shipments#tracking
```


-------------------
# How would an endpoint like this look currently?

-------------------
```ts
// utils/callApi.js
export const TMS_ENDPOINTS = {
  ...
  carrierNetwork: 'v2/shipper_carriers',
  ...
}

// apps/tms/types/carrierNetworks.ts
export interface ICarrierNetwork {
  id?: string
  name: string
  userEmail?: string
  onTimePickupPercentage?: number | null
  onTimeDeliveryPercentage?: number | null
  mcNumber?: string
  dotNumber?: string
  scac?: string
  lastUpdatedAt?: string
  rate?: string
  rateType?: 'line_haul' | 'all_in'
  carrierRank?: number | null
}

// apps/tms/carrier-network/utils/actions/carrierNetwork.ts
export const CREATE_CARRIER_ACTION_TYPES = generateApiActionTypes('CREATE_CARRIER')

const { CREATE_CARRIER_SUCCESS, CREATE_CARRIER_FAILURE } = CREATE_CARRIER_ACTION_TYPES

...

export const addCarrier = (shipperCarrier: ICarrierNetwork) => ({
  payload: {
    callApi: true,
    requestData: {
      endpoint: TMS_ENDPOINTS.carrierNetwork,
      method: 'POST',
      body: { shipperCarrier },
    },
    actionTypes: Object.keys(CREATE_CARRIER_ACTION_TYPES),
  },
  type: '',
})

// apps/tms/carrier-network/utils/reducers/carrierNetwork.ts
export const carrierNetworkDetailsReducer: Reducer<
  ICarrierNetworkDetailsState,
  CarrierNetworkActions
> = (state = defaultState, action) => {
  switch (action.type) {
    case CREATE_CARRIER_REQUEST:
      ...
    case CREATE_CARRIER_FAILURE:
      ...
    case CREATE_CARRIER_SUCCESS:
      ...
    default:
      return state
}

// apps/tms/carrier-network/utils/selectors/carrierNetwork.ts
import { IStore } from 'apps/tms/utils/reducers'

export const selectorCarrierNetworkDetails = (state: IStore) => state.carrier

export const carrierListStateSelector = (state: IStore) => state.carriers
```

-------------------

The important thing you'll notice being generated is the types associated with the API request and response.
```ts
export interface UpdateTrackingResponse {
  shipment?: ShipmentDetail
}

export interface UpdateTrackingPathParams {
  id: string
}

export interface UpdateTrackingRequestBody {
  /**
   * the state of the shipment
   */
  state: 'dispatched' | 'at_pickup' | 'in_transit' | 'at_delivery' | 'delivered'
  /**
   * id of the stop the update is for. Current stop sequence will be derived from this.
   */
  stopId?: number
  /**
   * ETA in UTC
   */
  eta?: string
}
```

-------------------

Everything else is boilerplate

```ts
export type UpdateTrackingProps = Omit<
  MutateProps<
    UpdateTrackingResponse,
    unknown,
    void,
    UpdateTrackingRequestBody,
    UpdateTrackingPathParams
  >,
  'path' | 'verb'
> &
  UpdateTrackingPathParams

/**
 * Sends updated tracking information
 */
export const UpdateTracking = ({ id, ...props }: UpdateTrackingProps) => (
  <Mutate<
    UpdateTrackingResponse,
    unknown,
    void,
    UpdateTrackingRequestBody,
    UpdateTrackingPathParams
  >
    verb='POST'
    path={`/v2/shipments/${id}/tracking`}
    {...props}
  />
)

export type UseUpdateTrackingProps = Omit<
  UseMutateProps<
    UpdateTrackingResponse,
    unknown,
    void,
    UpdateTrackingRequestBody,
    UpdateTrackingPathParams
  >,
  'path' | 'verb'
> &
  UpdateTrackingPathParams

/**
 * Sends updated tracking information
 */
export const useUpdateTracking = ({ id, ...props }: UseUpdateTrackingProps) =>
  useMutate<
    UpdateTrackingResponse,
    unknown,
    void,
    UpdateTrackingRequestBody,
    UpdateTrackingPathParams
  >(
    'POST',
    (paramsInPath: UpdateTrackingPathParams) => `/v2/shipments/${paramsInPath.id}/tracking`,
    { pathParams: { id }, ...props }
  )
```

-------------------

```ts
/**
 * Sends updated tracking information
 */
export const useUpdateTracking = ({ id, ...props }: UseUpdateTrackingProps) =>
  useMutate<
    UpdateTrackingResponse,
    unknown,
    void,
    UpdateTrackingRequestBody,
    UpdateTrackingPathParams
  >(
    'POST',
    (paramsInPath: UpdateTrackingPathParams) => `/v2/shipments/${paramsInPath.id}/tracking`,
    { pathParams: { id }, ...props }
  )
```

------------------

Use in component
```ts
// All of this code is automatically generated!
import {
  useUpdateTracking,
  UpdateTrackingRequestBody,
} from 'apps/tms/types/apiTypes'

export const TrackingComponent = () => {
  const { mutate: updateTracking, error: updateTrackingError } = useUpdateTracking({
    id: shipmentInfo?.id ?? '',
  })

  ...

  // button handler implementation
  const submitTrackingUpdateHandler = async (editTrackingData: UpdateTrackingRequestBody) => {
    // Raises an exception if the request fails, meaning that the modal will not
    // close. We can easily show an error message in the modal by relying on the
    // error returned by the hook above.
    await updateTracking(editTrackingData)
    closeModal()
  }

  ...

  return (
    <>...</>
  )
}
```

-------------------

Use with React Hook Forms – short aside
```ts
const { handleSubmit, control, watch } = useForm<UpdateTrackingRequestBody, { isCarrierContext: boolean }>({
  resolver: yupResolver(editTrackingSchema),
  mode: 'onChange',
  context: { isCarrierContext }
})
```

-------------------

Writing more robust e2e tests
```ts
describe('Updating tracking from shipment details', () => {
  describe('when changing the status is successful', () => {
    it('shipment on the page should be updated', () => {
      cy.route('POST', '/v2/shipments/*/tracking', { shipment: trackingResponse }).as(
        'postShipmentTracking'
      )

      /**
       * Interactions with the UI
       */
      cy.clickSelectOption("[data-test='tracking-status-functional-tag']", 'At Pickup')
      cy.selectFromDatePicker('.date-picker', moment())
      cy.dataTest('tracking-eta-time').type('11pm{enter}')

      // Press submit button
      cy.dataTest('confirm-edit-status').click()

      // Assertion on API request
      cy.wait('@postShipmentTracking')
        .its('request.body')
        .should('eql', {
          eta: `${moment().format('YYYY-MM-DD')}T23:00:00-05:00`,
          state: 'at_pickup',
          stopId: 2383,
        })
    })
  })
})
```

-------------------

What you can do?
- Tell your backend engineers to write restful-react compliant endpoints!
- Talk to me about setting up your app with restful-react
- Reread the TDR! – https://transfix.atlassian.net/wiki/spaces/FE/pages/1038680065/API+Requests+with+restful-react

