# Orders - Order Repository Mapping

## Obiettivo

Riepilogo operativo del repository principale `OrderRepository` (lista, dettaglio, create, update, delete ordine).

## Operazioni coperte

- `getList` -> lista ordini
- `getSingle` -> dettaglio ordine
- `createSingle` -> inserimento ordine
- `updateSingle` -> aggiornamento ordine
- `deleteSingle` -> cancellazione ordine

## Parametri in ingresso (Presentation)

- Lista ordini (`getOrders`)
  - Query: `timeZone` (required), `fullTextFilter?`, `from?`, `to?`, `statuses?`, `typeIds?`, `statusIds?`, `channelTypeIds?`, `isExternal?`, `isAuthorizationNeeded?`, `isAuthorizationRejected?`, `customerId?`, `pharmacyChainId?`, `userId?`, `pageNumber?`, `pageItemCount?`.
- Singolo ordine (`getOrder`)
  - Path: `orderId`
  - Query: `entity` (`pending|closed|erp`), `timeZone`.
- Creazione (`postOrder`)
  - Body principale: campi testata.
- Aggiornamento (`putOrder`)
  - Path: `orderId`
  - Body analogo alla create (+ `valueDateId?`).
- Cancellazione (`deleteOrder`)
  - Path: `orderId`.

## Stored procedure usate

- Lista: `tb_erp_order_list_g_aidea`
- Dettaglio: `tb_erp_order_list_g`
- Inserimento: `tb_erp_order_i`
- Aggiornamento: `tb_erp_order_u`
- Cancellazione: `tb_erp_order_d`

## Parametri principali inviati alle SP

- `getList`
  - `filterSecurityOwnerId`, `filter`, `filterChainId`, `pageNumber`, `rowsOfPage`, `orderBy='OrderDate DESC'`, `fields`.
  - `filter` costruito da customer/type/status/channelType/statusCodes/fullText/date-range e flag autorizzazione/esterno.
- `getSingle`
  - `filter = ErpOrderId='<id>'`, `includeAncestors`, `addHistory`, `fields`.
- `createSingle`
  - Dati ordine + riferimenti (`clientId`, address ids, status/type/payment/currency, sconti, condition type/deadline, channelType).
- `updateSingle`
  - Aggiornamento campi ordine e riferimenti (include `valueDateId`).
- `deleteSingle`
  - `ErpOrderId`.

## Mapping dati in uscita (repository -> entity)

Esempio (estratto semplificato di `mapToEntity`):

```ts
return new Order({
  id: external.ErpOrderId,
  customer: new Customer({
    id: external.ClientId,
    code: external.ClientAccountCode,
    name: external.ClientName,
    email: external.ClientEmail ?? external.ClientEmail2,
  }),
  delivery: external.CustomerAddressIdDelivery
    ? new Customer({
        id: external.CustomerAddressIdDelivery,
        name: external.DeliveryName,
      })
    : null,

  status: new OrderStatus({
    id: external.OrderStatusId,
    code: external.OrderStatusCode,
    name: external.OrderStatusCodeName,
    abbr: external.OrderStatusCodeAbbr,
  }),
  type: new OrderType({
    id: external.OrderTypeId,
    code: external.OrderTypeCode,
    name: external.OrderTypeCodeName,
    abbr: external.OrderTypeCodeAbbr,
  }),

  paymentMode: external.PaymentModeId
    ? new OrderPaymentMode({
        id: external.PaymentModeId,
        code: external.PaymentModeCode,
        name: external.PaymentModeCodeName,
      })
    : null,
  deferredPayment: external.DeferredPaymentId
    ? new OrderDeferredPayment({
        id: external.DeferredPaymentId,
        code: external.DeferredPaymentCode,
        name: external.DeferredPaymentCodeName,
      })
    : null,
  currency: external.CurrencyId
    ? new OrderCurrency({
        id: external.CurrencyId,
        code: external.CurrencyCode,
        name: external.CurrencyCodeName,
      })
    : null,

  date: external.OrderDate
    ? retailDateToJs(external.OrderDate, timeZone)
    : null,
  deliveryDate: external.ScheduledDeliveryDate
    ? retailDateToJs(external.ScheduledDeliveryDate, timeZone)
    : null,

  conditionType: external.ConditionTypeId
    ? new OrderConditionType({
        id: external.ConditionTypeId,
        code: external.ConditionTypeCode,
        name: external.ConditionTypeCodeName,
      })
    : null,
  conditionDeadline: external.ConditionDeadlineId
    ? new OrderConditionDeadline({
        id: external.ConditionDeadlineId,
        code: external.ConditionDeadlineCode,
        name: external.ConditionDeadlineCodeName,
      })
    : null,
  valueDate: external.ValueDateId
    ? new OrderValueDate({
        id: external.ValueDateId,
        code: external.ValueDateCode,
        name: external.ValueDateCodeName,
      })
    : null,
});
```

## Struttura risposta API

- Lista: `IEntityPaginatedList<Order>`
- Dettaglio: `Order`
- Create: `{ id }`
- Update: `204`
- Delete: `204`

## Note importanti

- Nel dettaglio, `includeAncestors` e `addHistory` sono disattivati solo quando `entity='erp'`.
- La ricerca full-text su lista usa `OrderRepository.searchFields` (campi codice/nome/stato/tipo/pagamento/canale) concatenando varie condizioni LIKE in OR.
