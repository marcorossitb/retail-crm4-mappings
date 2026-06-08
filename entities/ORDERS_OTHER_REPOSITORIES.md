# Orders - Other Repositories (Non Picklist)

## Obiettivo
Riepilogo sintetico dei repository `Orders` non-picklist (escluso `OrderRepository`, documentato a parte).

## OrderAuthorizationRepository
- Operazione: `getList`.
- Input principale: `order.id`.
- Stored procedure: `tb_erp_order_authorization_list_g`.
- Parametri SQL principali: `filter = ErpOrderId='<orderId>'`, `fields`.
- Output: `OrderAuthorization` con status, note, date request/manage e utenti (`applicant`, `licensing`, `rejector`, `authority`).

Esempio mapping output:
```ts
{
  items: OrderAuthorization[]; // array di autorizzazioni dell'ordine
}

new OrderAuthorization({
  id: external.ErpOrderAuthorizationId,
  isAuthorizationNeeded: Boolean(external.IsAuthorizationNeeded),
  isAuthorizationRejected: Boolean(external.IsAuthorizationRejected),
  note: external.NoteText,
  customer: new Customer({ id: external.ClientId, name: external.ClientName }),
  status: new OrderAuthorizationStatus({
    id: external.StatusId,
    code: external.StatusCode,
    name: external.StatusCodeName,
    abbr: external.StatusCodeAbbr,
  }),
});
```

## OrderWarningRepository
- Operazione: `getList`.
- Input principale: `order.id`.
- Stored procedure: `tb_erp_order_check_g`.
- Parametri SQL principali: `erpOrderId`, `fields`, `orderBy` custom per severita (`ERR`, `WAR`, `INF`).
- Output: `OrderWarning` con `type`, `checkType`, `origin`, `description`, `licensing`.
Esempio mapping output:

```ts
{
  items: OrderWarning[]; // array di warning dell'ordine
}

new OrderWarning({
  id: external.ErpOrderCheckId,
  type: new OrderWarningType({
    id: external.CheckInformationTypeId,
    code: external.CheckInformationTypeCode,
    name: external.CheckInformationTypeCodeName,
  }),
  checkType: new OrderWarningCheck({
    id: external.ErpOrderCheckTypeId,
    code: external.ErpOrderCheckTypeCode,
    name: external.ErpOrderCheckTypeCodeName,
  }),
  origin: external.OriginDescription,
  description: external.Description,
});
```

## OrderNoteRepository
- Operazioni: `getList`, `createSingle`, `updateSingle`, `deleteSingle`.
- Stored procedure:
  - lista: `tb_note_list_g`
  - create: `tb_note_i`
  - update: `tb_note_u`
  - delete: `tb_note_d`
- Parametri SQL principali:
  - lista: `Entity='erp_order'` + `ParentId` (o filtro sicurezza su ordini accessibili)
  - create/update: `noteId`, `text`, `noteTypeId`, `date`
- Output: `OrderNote` con `type` (`OrderNoteType`) e `date`.

Esempio mapping output:
```ts
{
  items: OrderNote[]; // array di note dell'ordine
}

new OrderNote({
  id: external.NoteId,
  orderId: external.ParentId,
  text: external.Text,
  type: new OrderNoteType({
    id: external.NoteTypeId,
    code: external.NoteTypeCode,
    name: external.NoteTypeCodeName,
    abbr: external.NoteTypeCodeAbbr,
  }),
  date: new Date(external.Date),
});
```

## OrderRowRepository
- Operazioni: `getList`, `createList`, `updateList`, `deleteList`.
- Stored procedure:
  - lista: `tb_erp_order_detail_list_g`
  - create row: `tb_erp_order_detail_i`
  - update row: `tb_erp_order_detail_u`
  - delete row: `tb_erp_order_detail_d`
- Parametri SQL principali:
  - lista: `filter` su `ErpOrderId` (o security fallback), `fields`, `addHistory=1`, `orderBy='ProductCode'`
  - create/update: dettagli riga prodotto (price/pricelist/qty/discount/promotion/deferred payment/scheduledDeliveryDate)
  - delete: `erpOrderDetailId`, `applyLogics`
- Output: `OrderRow` con `product` (`OrderProduct` completo) + `status` (`OrderRowStatus`) + `totalPrice`.
- Nota: create/update/delete sono transazionali e applicano `applyLogics=true` sull'ultima riga del batch.

Esempio mapping output:
```ts
{
  items: OrderRow[]; // array di righe dell'ordine
}

new OrderRow({
  id: external.ErpOrderDetailId,
  orderId: external.ErpOrderId,
  product: new OrderProduct({
    id: external.ProductId,
    code: external.ProductCode,
    name: external.ProductCodeName,
    publicPrice: external.Price,
    quantity: external.Quantity,
    pricelist: external.PricelistId
      ? new OrderProductPricelist({ id: external.PricelistId, name: external.PricelistCodeName })
      : undefined,
    promotion: external.SpecialOfferId
      ? new OrderProductPromotion({ id: external.SpecialOfferId, name: external.SpecialOfferDescription })
      : undefined,
  }),
  status: new OrderRowStatus({
    id: external.OrderDetailStatusId,
    code: external.OrderDetailStatusCode,
    name: external.OrderDetailStatusCodeName,
    abbr: external.OrderDetailStatusCodeAbbr,
  }),
  totalPrice: external.TotalPrice,
});
```

## Product & Pricing Repositories

### OrderBrandRepository
- Operazione: `getList`.
- Stored procedure: `tb_product_list_g_standard`.
- Parametri SQL: `filter="Entity='erp_brand'"`, `orderBy='ProductCodeName'`, `date`, `fields`.
- Output: `OrderBrand` (`id`, `code`, `name`).

Esempio mapping output:
```ts
{
  items: OrderBrand[]; // array di brand disponibili
}
new OrderBrand({
  id: external.ProductId,
  code: external.ProductCode,
  name: external.ProductCodeName,
});
```

### OrderProductRepository
- Operazione: `getList`.
- Input principali: `order.id`, filtri opzionali `promotionId`, `brandId`, `brandIds[]`, `timeZone`.
- Stored procedure: `tb_erp_order_pricelist_g`.
- Parametri SQL: `erpOrderId`, `filter`, `fields`, `orderBy='ProductCodeName'`.
- Output: `OrderProduct` ricco (prezzi, sconti, flag editabilita, promo, pricelist, deferred payment, brand, segmentazioni, date).

Esempio mapping output:
```ts
{
  items: OrderProduct[]; // array di prodotti dell'ordine
}

new OrderProduct({
  id: external.ProductId,
  code: external.ProductCode,
  name: external.ProductCodeName,
  publicPrice: external.Price,
  quantity: external.Quantity,
  isEnabledPrice: Boolean(external.IsEnabledPrice),
  pricelist: external.PricelistId
    ? new OrderProductPricelist({ id: external.PricelistId, name: external.PricelistCodeName })
    : undefined,
  deferredPayment: external.DeferredPaymentId
    ? new OrderDeferredPayment({ id: external.DeferredPaymentId, code: external.DeferredPaymentCode, name: external.DeferredPaymentCodeName })
    : undefined,
  brand: external.BrandId
    ? new OrderBrand({ id: external.BrandId, code: external.BrandCode, name: external.BrandCodeName })
    : undefined,
});
```

### OrderProductPricelistRepository
- Operazione: `getList`.
- Input: `order.id`, `product.id`.
- Stored procedure: `tb_erp_order_pricelist_list_g`.
- Parametri SQL: `filter = ErpOrderId AND ProductId`, `fields`.
- Output: `OrderProductPricelist`.
Esempio mapping output:
```ts
new OrderProductPricelist({
  id: external.PricelistId,
  name: external.PricelistCodeName,
  productId: external.ProductId,
});
```

### OrderProductPricelistDetailRepository
- Operazione: `getList`.
- Input: `product.id`, `product.pricelist.id`.
- Stored procedure: `tb_pricelist_product_list_g`.
- Parametri SQL: `filter = PriceListId AND ProductId`, `fields`.
- Output: `OrderProductPricelistDetail` (`minQuantity`, `maxQuantity`, `maxDiscount`, `deferredPaymentCodeName`).

Esempio mapping output:
```ts
{
  items: OrderProductPricelistDetail[]; // array di pricelist details per il prodotto
}

new OrderProductPricelistDetail({
  id: external.PricelistId,
  name: `${external.PricelistCodeName} (${external.ProductCode} ${external.ProductCodeName})`,
  productId: external.ProductId,
  minQuantity: external.MinQuantity,
  maxQuantity: external.MaxQuantity,
  maxDiscount: external.DiscountMax,
  deferredPaymentCodeName: external.DeferredPaymentCodeName,
});
```

### OrderPromotionRepository
- Operazione: `getList`.
- Input: `order.id`.
- Stored procedure: `tb_erp_specialoffer_product_list_g`.
- Parametri SQL: `filter = ErpOrderId`, `orderBy='SpecialOfferDescription'`, `fields`.
- Output: `OrderPromotion` (`id`, `name`, `isNotApplicable`).

Esempio mapping output:
```ts
{
  items: OrderPromotion[]; // array di promozioni per l'ordine
}

new OrderPromotion({
  id: external.SpecialOfferId,
  name: external.SpecialOfferDescription,
  isNotApplicable: Boolean(external.IsNotApplicable),
});
```

### OrderProductPromotionRepository
- Operazione: `getList`.
- Input: `order.id`, `product.id`.
- Stored procedure: `tb_erp_specialoffer_product_list_g`.
- Parametri SQL: `filter = ErpOrderId AND ProductId`, `fields`.
- Output: `OrderProductPromotion`.

Esempio mapping output:


```ts
{
  items: OrderProductPromotion[]; // array di promozioni per il prodotto dell'ordine
}

new OrderProductPromotion({
  id: external.SpecialOfferId,
  name: external.SpecialOfferDescription,
  productId: external.ProductId,
});
```

## Action Repositories

### OrderCloneRepository
- Operazione: `createSingle`.
- Stored procedure: `tb_erp_order_clone` (con output `NewErpOrderId`).
- Parametri SQL principali: `ErpOrderId`, `OrderTypeId`, `IsCopyNote`, `IsReapplyLogics`, delivery/payment/currency/condition/address/leader.
- Output: `NewErpOrderId`.
Esempio output:
```ts
const newOrderId = request.output.NewErpOrderId; // string
```

### OrderValidationRepository
- Operazione: `createSingle`.
- Stored procedure: `tb_erp_order_setconvalidation`.
- Parametro SQL: `erpOrderId`.
- Output: nessun payload (`void`).

### OrderVerificationRepository
- Operazione: `createSingle`.
- Stored procedure: `tb_erp_order_check_i`.
- Parametro SQL: `erpOrderId`.
- Output: nessun payload (`void`).

## Riferimenti API (Presentation)
- Authorizations: `getOrderAuthorizations.ts`
- Warnings: `getOrderWarnings.ts`
- Notes: `getOrderNotes.ts`, `postOrderNote.ts`, `putOrderNote.ts`, `deleteOrderNote.ts`
- Rows: `getOrderRows.ts`, `postOrderRows.ts`, `putOrderRows.ts`, `deleteOrderRows.ts`
- Products/Pricing/Promotions/Brand: `getOrderProducts.ts`, `getOrderProductPricelists.ts`, `getOrderProductPricelistDetails.ts`, `getOrderPromotions.ts`, `getOrderProductPromotions.ts`, `getOrderBrands.ts`
- Actions: `postOrderClone.ts`, `postOrderValidations.ts`, `postOrderVerifications.ts`
