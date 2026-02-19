# Customers - Customer Connections Mapping

## Obiettivo

Chiamata per ottenere le connessioni di un customer (altri customer) con i relativi dettagli (tipo connessione, campi estesi del nodo collegato, ecc.).

## Operazioni API coperte

- `GET /v1/customers/connections`
- `GET /v1/customers/:customerId/connections`
- `POST /v1/customers/:customerId/connections`
- `PUT /v1/customers/:customerId/connections/:connectionId`
- `DELETE /v1/customers/:customerId/connections/:connectionId`

## Parametri in ingresso (Presentation)

- `GET .../connections`: path `customerId?` opzionale, query
- `entity?`, `onlyChildren?`, `fullTextFilter?`, `clusterIds?`, `clusterTypeIds?`, `businessRoleIds?`, `customerBlock?`, `deliveryBlockIds?`, `statusIds?`, `isInPOAId?`, `getChildFullFields?`, `getParentFullFields?`, `pageNumber?`, `pageItemCount?`
- `POST .../connections`: path `customerId`, body `toId`, `typeId`
- `PUT .../connections/:connectionId`: path `customerId`, `connectionId`, body `isDefault?`
- `DELETE .../connections/:connectionId`: path `customerId`, `connectionId`

## Stored procedure usate

- Lista connessioni: `tb_account_relationship_list_g`
- Lookup tipi connessione: `tb_picklist_list_g` (con filtro `Entity='account_r.entity'`)
- Lookup customer (solo in alcuni casi): `tb_account_list_g`
- Inserimento connessione: `tb_account_r_i`
- Aggiornamento connessione: `tb_account_r_u`
- Cancellazione connessione: `tb_account_r_d`

## Parametri principali inviati alle stored procedure

- `getList` su `tb_account_relationship_list_g`
- `filter` composto dinamicamente (customer id, entity, fullText, ruoli, blocchi, cluster, status, POA)
- `fields` variabili in base a `getChildFullFields` e `getParentFullFields`
- `isChildFullField`, `isParentFullField`, `orderBy='ParentName, ChildName'`, `minRowNum`, `maxRowNum`
- `createSingle` su `tb_account_r_i`: `accountRelationshipId`, `parentAccountId`, `childAccountId`, `entity`, `validFrom`
- `updateSingle` su `tb_account_r_u`: `accountRelationshipId`, `isDefault`
- `deleteSingle` su `tb_account_r_d`: `accountRelationshipId`

## Mapping dati in uscita (repository -> entity)

Esempio (estratto semplificato di `mapToEntity`):

```ts
return new CustomerConnection({
  id: external.AccountRelationshipId,
  entity: external.Entity,
  isDefault: Boolean(external.IsDefault) ?? false,

  type: !external.Entity
    ? undefined
    : new CustomerConnectionType({
        id: external.Entity,
        code: external.Entity,
        name: typesMap[external.Entity], // da lookup tb_picklist_list_g
      }),

  from: !external.ParentAccountId
    ? undefined
    : new Customer({
        id: external.ParentAccountId,
        name: external.ParentName,
        businessRole: external.ParentMS_AccountCustomerType,
        deliveryBlock: external.ParentDeliveryBlockId
          ? new CustomerDeliveryBlock({
              id: external.ParentDeliveryBlockId,
              code: external.ParentDeliveryBlockCode,
              name: external.ParentDeliveryBlockCodeName,
              status: external.ParentDeliveryBlockValue1,
            })
          : undefined,
      }),

  to: !external.ChildAccountId
    ? undefined
    : new Customer({
        id: external.ChildAccountId,
        code: external.ChildAccountCode,
        name: external.ChildName,
        businessRole: external.ChildMS_AccountCustomerType,
        deliveryBlock: external.ChildDeliveryBlockId
          ? new CustomerDeliveryBlock({
              id: external.ChildDeliveryBlockId,
              code: external.ChildDeliveryBlockCode,
              name: external.ChildDeliveryBlockCodeName,
              status: external.ChildDeliveryBlockValue1,
            })
          : undefined,
      }),
});
```

Quando `getChildFullFields` / `getParentFullFields` sono `true`, nei nodi `to`/`from` vengono valorizzati anche i campi estesi (geografia, cluster, clusterType, POA, externalDeliveryBlock, ecc.).

## Struttura risposta API

- `GET .../connections`: `IEntityPaginatedList<CustomerConnection>`
- `POST .../connections`: `{ id }`
- `PUT .../connections/:connectionId`: `204` body vuoto
- `DELETE .../connections/:connectionId`: `204` body vuoto

## Note importanti

- Se `onlyChildren=true` e `entity!='TransferOrder'`, il repository aggiunge anche una riga sintetica `ME` per il customer corrente.
- In `createSingle`, per `type.id='chronicle'` viene bloccata la creazione se esiste gia una connessione `chronicle` sul parent (`Can not add more than one chronicle`).
