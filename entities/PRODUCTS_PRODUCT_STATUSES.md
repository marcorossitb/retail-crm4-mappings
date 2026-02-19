# Products - Product Status Repository Mapping

## Obiettivo

Riepilogo operativo dell'entita `ProductStatus` usata dal frontend per popolare la picklist degli stati prodotto.

## Parametri in ingresso (Presentation)

- Path params: nessuno
- Body: nessuno
- Query: nessuna

## Stored procedure usata

- `tb_picklist_list_g`

## Parametri principali inviati alla SP

- `filter="Entity = 'erp_product_status'"`
- `orderBy='SortOrder'`
- `fields='SortOrder, PicklistId, ItemCode, ItemCodeName'`
- Parametri tecnici SQL da `RetailConnection`: `SystemUserId`, `LineCode`, `RoleName`

## Esempio mapping output

```ts
new ProductStatus({
  id: external.PicklistId,
  code: external.ItemCode,
  name: external.ItemCodeName,
});
```

## Struttura risposta API

- `IEntityPaginatedList<ProductStatus>`:
  - `items[]` con campi `id`, `code`, `name`
  - `pagination.itemCountGlobal`
  - `pagination.itemCountAfterFilterAndPagination`
