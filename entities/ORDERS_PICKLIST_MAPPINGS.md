# Orders Picklist Mappings (AIDEA 1_0_0_0)

## Obiettivo
Documentare le entita picklist usate dal frontend nell'area Orders.

## Contratto comune
- Stored procedure principale: `tb_picklist_list_g`
- Parametri tecnici SQL sempre iniettati da `RetailConnection`: `SystemUserId`, `LineCode`, `RoleName`
- Struttura risposta repository: `IEntityPaginatedList<T>`

Mappatura piu comune (quasi tutte le entita):
- `id <- PicklistId`
- `code <- ItemCode`
- `name <- ItemCodeName`
- `abbr <- ItemCodeAbbr`
- `isDefault <- IsDefault`

## Entita picklist (`tb_picklist_list_g`)
- `OrderAuthorizationStatus`
  - Filtro: `Entity = 'erp_order_authorization_status'`
  - Note: mappatura standard.

- `OrderChannel`
  - Filtro: `Entity = 'channel'`
  - Note: mappatura standard.

- `OrderChannelType`
  - Filtro: `Entity = 'erp_datacollectionorigin'`
  - Note: mappatura standard.

- `OrderCurrency`
  - Filtro: `Entity = 'currency'`
  - Ordinamento: `IsDefault DESC, ItemCodeName`
  - Note: mappatura standard.

- `OrderDeferredPayment`
  - Filtro: `ChildEntity='DeferredPayment' AND PicklistId='<paymentModeId>'`
  - Ordinamento: `ChildIsDefault DESC, ChildItemCodeName, ChildItemCode`
  - Note: mappatura non standard con campi `Child*`.
  - Mapping: `id <- ChildPicklistId`, `code <- ChildItemCode`, `name <- ChildItemCodeName`, `abbr <- ChildItemCodeAbbr`, `isDefault <- ChildIsDefault`.

- `OrderNoteType`
  - Filtro: `Entity = 'notetype'`
  - Note: mappatura standard.

- `OrderPaymentMode`
  - Filtro: `Entity = 'paymentmode'`
  - Note: mappatura standard.

- `OrderRowStatus`
  - Filtro: `Entity = 'erp_orderdetailstatus'`
  - Note: mappatura standard.

- `OrderStatus`
  - Filtro: `Entity = 'erp_orderstatus'`
  - Ordinamento: `ItemCodeName`
  - Note: mappatura standard.

- `OrderType`
  - Filtro: `Entity = 'erp_ordertype'`
  - Note: mappatura standard.

- `OrderValueDate`
  - Filtro: `Entity = 'erp_valuedate'`
  - Ordinamento: `SortOrder`
  - Note: mappatura standard; `SortOrder` viene usato per ordinamento, non esposto nell'entita.

- `OrderWarningCheck`
  - Filtro: `Entity = 'erp_orderchecktype'`
  - Note: mappatura standard.

- `OrderWarningType`
  - Filtro: `Entity = 'erp_check_informationtype'`
  - Note: mappatura standard.

## Parametri API rilevanti
- Tutte le route picklist hanno `params/body/query` vuoti, tranne:
- `getOrderDeferredPayments`: richiede `paymentModeId` in query.
