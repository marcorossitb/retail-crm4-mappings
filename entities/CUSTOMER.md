# Customers - Customer Entity Mapping

## Obiettivo
Riepilogo operativo dell'entita `Customer` usata dal frontend.

## Operazioni API coperte
- `GET /v1/customers` -> lista clienti
- `GET /v1/customers/:customerId` -> dettaglio cliente
- `POST /v1/customers` -> creazione cliente
- `PUT /v1/customers/:customerId` -> aggiornamento cliente

## Parametri in ingresso (Presentation)
- `GET /v1/customers`: query supportata
- `fullTextFilter?`, `clusterIds?`, `brickIds?`, `customerClassIds?`, `pageNumber?`, `pageItemCount?`, `contactId?`, `onlyInTarget?`, `businessRoleIds?`, `customerBlock?`, `deliveryBlockIds?`, `maxiGroupIds?`, `clusterTypeIds?`, `userId?`, `connectionType?`, `connectionCustomerId?`, `accountIds?`, `statusIds?`, `accountStatusIds?`, `statusCodes?`, `isInPOAId?`, `withParent?`, `getPharmacyChains?`, `isPharmacyChainFullTextSearchMode?`, `withIban?`
- `GET /v1/customers/:customerId`: path `customerId`, query `isParent?`
- `POST /v1/customers`: body con i campi anagrafici/commerciali principali (es. `code`, `externalCode`, `name`, `businessRole`, riferimenti geografici, fiscali, pagamento, cluster, parent)
- `PUT /v1/customers/:customerId`: path `customerId`, body analogo al `POST`

## Stored procedure usate
- Lettura lista e dettaglio: `tb_account_list_g`
- Creazione: `tb_account_i`
- Post-creazione (compilazioni): `tb_pricelist_account_compile_single`, `tb_special_offer_account_compile_single`, `tbbatch_tbden_erp_account_product_price`
- Aggiornamento: `tb_account_u`

## Parametri principali inviati alle stored procedure
- Lettura lista: `callFrom='OTC'`, `fields=<databaseFields con placeholder lineCode>`, `orderBy='Name'`, `minRowNum`, `maxRowNum`, `Filter` composto da più condizioni
- Lettura singolo: `filter = AccountId='<id>'` oppure `ParentId='<id>'` quando `isParent=true`
- Creazione (`tb_account_i`): mapping completo del payload customer su parametri SQL (`AccountCode`, `ExternalCode1`, `Name`, `MS_AccountCustomerType`, `AccountCategoryCode`, campi indirizzo/contatto/fiscali/pagamento/cluster)
- Aggiornamento (`tb_account_u`): stessi macro-campi della create + lookup preliminare di `Address1_AddressId` su `tb_account_list_g`

## Mapping dati in uscita (repository -> entity)
Esempio (estratto semplificato di `mapToEntity`):

```ts
return new Customer({
  id: external.AccountId,
  code: external.AccountCode,
  name: external.Name,
  externalCode: external.ExternalCode1,
  erpCode: external.ErpCode,

  customerBlock: Boolean(external.IsAccountBlockedReason),
  deliveryBlock: external.DeliveryBlockId
    ? new CustomerDeliveryBlock({
        id: external.DeliveryBlockId,
        code: external.DeliveryBlockCode,
        name: external.DeliveryBlockCodeName,
        status: external.DeliveryBlockValue1,
      })
    : undefined,

  type: external.AccountCategoryCRMCode
    ? new CustomerType({
        id: external.AccountCategoryCRMCode.toString(),
        code: external.AccountCategoryCode,
        name: external.AccountCategoryCodeName,
      })
    : undefined,
  status: external.ErpStatusId
    ? new CustomerStatus({
        id: external.ErpStatusId,
        code: external.ErpStatusCode,
        name: external.ErpStatusCodeName,
      })
    : undefined,

  town: external.Address1_TownId
    ? new GeographyTown({
        id: external.Address1_TownId,
        code: external.Address1_TownCode,
        name: external.Address1_TownCodeName,
      })
    : undefined,
  street: external.Address1_Line1,

  companyType: external.CompanyTypeId
    ? new CustomerCompanyType({
        id: external.CompanyTypeId,
        code: external.CompanyTypeCode,
        name: external.CompanyTypeCodeName,
      })
    : undefined,
  paymentMode: external.PaymentModeId
    ? new CustomerPaymentMode({
        id: external.PaymentModeId,
        code: external.PaymentModeCode,
        name: external.PaymentModeCodeName,
      })
    : undefined,

  parent: external.ParentId
    ? new Account({
        id: external.ParentId,
        name: external.AccountNameParent,
      })
    : undefined,
  isInPOA: external.Segmentation1Code == 'PL',
  isInPOAId: external.SegmentationID1,
  cluster: external.AccountClusterId
    ? new CustomerCluster({
        id: external.AccountClusterId,
        code: external.AccountClusterCode,
        name: external.AccountClusterCodeName,
      })
    : undefined,
});
```

Altri campi presenti nella mappatura completa: `CID`, `ImsCode`, `MaxValueCap`, `SellInYearObjective`, `IsPremiumCustomer`, `IsRequestAttention`, `MS_Accountcluster`, `MS_ConditionType_ItemCode`, `MS_ConditionDeadline_ItemCode`.

## Struttura risposta API
- `GET /v1/customers`: `IEntityPaginatedList<Customer>` con `items[]` e `pagination`
- `GET /v1/customers/:customerId`: singolo `Customer` (`toJSON()`) + `insights: []`
- `POST /v1/customers`: `{ id }`
- `PUT /v1/customers/:customerId`: `204` body vuoto

## Note importanti
- In `createSingle` c'e una transazione esplicita con rollback globale in caso errore.
- Le compilazioni post-create vengono eseguite subito dopo `tb_account_i`.
- Alcuni filtri presenti nella query presentation (`customerClassIds`, `brickIds`, `contactId`, `userId`) non risultano applicati in `getCustomerListFilters` del repository.
