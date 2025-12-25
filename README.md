# readme-send-money-base-promptVoici la **conversion complète du prompt en `README.md`**, prête à être déposée telle quelle à la racine du projet.
Tout le contenu est conservé, structuré et formaté en **Markdown propre**.

---

````md
# SendMoney Inbound – Digipay → MTN MoMo (RequestToPay)

## 🎯 Objectif

Implémenter un **flux INBOUND SendMoney** permettant de créditer un wallet Digipay via **MTN MoMo (RequestToPay)**, **sans créer de nouvelle intégration partenaire**.

> **SendMoney (Mobile App Digipay) → INBOUND → Wallet MTN MoMo (RequestToPay)**

L’implémentation est :
- propre
- structurée
- exploitable en production
- extensible (multi-wallets)

---

## 🧠 RÈGLE D’OR

- **SendMoney = channel Digipay**
- **MTN MoMo = wallet externe**
- **INBOUND = CREDIT Digipay**
- On **réutilise MOMO (RequestToPay)** existant :
  - payload différent  
  - logique métier différente  
  - nommage différent  

❌ Pas de duplication bête  
✅ Une seule intégration MTN MoMo

---

## 🗂️ 1️⃣ Arborescence – SendMoney Inbound

Architecture **parallèle exacte de `momo/`**, mais orientée **channel**.

```text
com.digipay.app.digipartner.main.rest.v1.sendmoney
│
├── inboundFeature
│   └── SendMoneyInboundRestApi.java
│
├── implementation
│   └── SendMoneyInboundInterfaceImpl.java
│
├── interfaces
│   └── SendMoneyInboundInterface.java
│
├── request
│   └── SendMoneyInboundRequest.java
│
├── response
│   └── SendMoneyInboundResponse.java
│
├── validation
│   └── SendMoneyValidation.java
│
└── utils
    └── SendMoneyUtils.java
````

📌 **MTN MoMo reste dans `momo/`**
➡️ On l’appelle, **on ne le réécrit pas**.

---

## 🔁 2️⃣ Flow Global – INBOUND SendMoney

```text
SendMoney App
   |
   |  POST /sendmoney/inbound
   |
SendMoneyInboundRestApi
   |
   |  validation + mapping
   |
SendMoneyInboundInterfaceImpl
   |
   |  RequestToPay (MTN MoMo)
   |
MTN MoMo
   |
   |  202 Accepted
   |
CheckStatus / Callback
   |
DigipartnerTransaction (CREDIT)
```

---

## 🧾 3️⃣ DTO – Payload SendMoney Inbound

Inspiré de la capture Postman.

### `SendMoneyInboundRequest.java`

```java
package com.digipay.app.digipartner.main.rest.v1.sendmoney.request;

import lombok.Data;
import java.io.Serializable;

@Data
public class SendMoneyInboundRequest implements Serializable {

    // Meta
    private String intent; // "inc_to_wallet"
    private String requestType; // "credit"
    private String channel; // SEND_MONEY_APP

    // Amount
    private Double senderAmount;
    private Double beneficiaryAmount;
    private String senderCurrency;
    private String beneficiaryCurrency;

    // Beneficiary (wallet owner)
    private String beneficiaryPhoneNumber;
    private String beneficiaryFirstName;
    private String beneficiaryLastName;
    private String beneficiaryCountry;

    // Sender (payer)
    private String senderFirstName;
    private String senderLastName;
    private String senderCountry;
    private String senderMobilePhone;

    // Business
    private String purpose;
    private String transactionReason;

    // References
    private String issuertrxref;
}
```

---

## 🔌 4️⃣ Interface Métier

### `SendMoneyInboundInterface.java`

```java
package com.digipay.app.digipartner.main.rest.v1.sendmoney.interfaces;

import javax.ws.rs.core.Response;
import java.io.InputStream;

public interface SendMoneyInboundInterface {

    Response initiateInbound(
            String digipayAccessToken,
            InputStream input,
            String xcountry
    );
}
```

---

## 🧠 5️⃣ Implémentation Métier (Cœur du Système)

### `SendMoneyInboundInterfaceImpl.java`

```java
package com.digipay.app.digipartner.main.rest.v1.sendmoney.implementation;

import com.digipay.app.digipartner.main.rest.v1.common.SimpleUser;
import com.digipay.app.digipartner.main.rest.v1.momo.interfaces.MomoInterface;
import com.digipay.app.digipartner.main.rest.v1.sendmoney.interfaces.SendMoneyInboundInterface;
import com.digipay.app.digipartner.main.rest.v1.sendmoney.request.SendMoneyInboundRequest;
import com.digipay.app.digipartner.main.rest.v1.momo.request.Request;
import com.digipay.app.digipartner.main.rest.v1.simbapay.Uat;
import com.fasterxml.jackson.databind.ObjectMapper;

import javax.inject.Inject;
import javax.ws.rs.core.Response;
import java.io.InputStream;

public class SendMoneyInboundInterfaceImpl implements SendMoneyInboundInterface {

    @Inject
    private MomoInterface momoInterface; // 🔥 Réutilisation existante

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public Response initiateInbound(String token, InputStream input, String xcountry) {

        // 1️⃣ Vérification du token Digipay
        SimpleUser user = Uat.doVerifyToken(token);
        if (user == null || user.getUserid() == null) {
            return Response.status(Response.Status.UNAUTHORIZED)
                    .entity("Invalid Digipay token")
                    .build();
        }

        // 2️⃣ Mapping SendMoney → MTN MoMo
        SendMoneyInboundRequest smRequest;
        try {
            smRequest = mapper.readValue(input, SendMoneyInboundRequest.class);
        } catch (Exception e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity("Invalid payload")
                    .build();
        }

        // 3️⃣ Construction RequestToPay MTN
        Request momoRequest = new Request();
        momoRequest.setMobilePhoneNumber(smRequest.getSenderMobilePhone());
        momoRequest.setPhoneCountryCode("242");
        momoRequest.setAmount(smRequest.getSenderAmount());
        momoRequest.setCurrency(smRequest.getSenderCurrency());
        momoRequest.setIssuertrxref(smRequest.getIssuertrxref());
        momoRequest.setRequestType("SEND_MONEY_INBOUND");
        momoRequest.setDestinationWalletName("MTN_MOMO");
        momoRequest.setSourceId("SEND_MONEY_APP");

        // 4️⃣ Appel MTN MoMo (RequestToPay)
        return momoInterface.InitiateMomoPayement(
                token,
                Uat.convertObjectToStream(momoRequest),
                xcountry
        );
    }
}
```

📌 **Important**

* `SendMoneyInboundInterfaceImpl` **ne gère pas HTTP**
* Il **orchestre uniquement la logique métier**

---

## 🌐 6️⃣ REST API – SendMoney Inbound

### `SendMoneyInboundRestApi.java`

```java
package com.digipay.app.digipartner.main.rest.v1.sendmoney.inboundFeature;

import com.digipay.app.digipartner.main.rest.v1.sendmoney.interfaces.SendMoneyInboundInterface;
import com.digipay.app.digipartner.main.rest.v1.util.ApplicationConfig;

import javax.ejb.Asynchronous;
import javax.ejb.Stateless;
import javax.inject.Inject;
import javax.ws.rs.*;
import javax.ws.rs.container.AsyncResponse;
import javax.ws.rs.container.Suspended;
import javax.ws.rs.core.MediaType;
import java.io.InputStream;
import java.util.concurrent.TimeUnit;

@Path("api/v1/sendmoney/inbound")
@Stateless
public class SendMoneyInboundRestApi extends ApplicationConfig {

    @Inject
    private SendMoneyInboundInterface sendMoneyInboundInterface;

    @POST
    @Path("/wallet/mtn")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @Asynchronous
    public void inboundToMtn(
            @Suspended AsyncResponse asyncResponse,
            @HeaderParam("digipay-access-token") String token,
            @HeaderParam("xcountry") String xcountry,
            InputStream input
    ) {
        asyncResponse.setTimeout(90, TimeUnit.SECONDS);
        asyncResponse.resume(
                sendMoneyInboundInterface.initiateInbound(token, input, xcountry)
        );
    }
}
```

---

## 🧾 7️⃣ Gestion Automatique Existante

| Élément          | Composant                |
| ---------------- | ------------------------ |
| Token Digipay    | `Uat.doVerifyToken`      |
| RequestToPay MTN | `MomoInterface`          |
| Check Status     | Déjà existant            |
| Callback         | Déjà existant            |
| Ledger CREDIT    | `DigipartnerTransaction` |

---

## ✅ Résultat Final

✔ SendMoney **n’est pas un partenaire**
✔ Une seule intégration MTN MoMo
✔ Architecture **claire, scalable, FinTech-ready**
✔ Extension simple vers :

* Orange Money
* Wave
* Airtel

➡️ **Sans modifier SendMoney**

---

## 🚀 Prochaines Étapes (au choix)

1️⃣ Implémenter le **mapping DB DigipartnerTransaction (CREDIT)**
2️⃣ Ajouter un **anti-duplicate `issuertrxref` côté SendMoney**
3️⃣ Factoriser un **Inbound Engine commun**
4️⃣ Créer un **diagramme de séquence officiel** (audit / régulateur)

👉 Indique le **numéro**, on continue 💼🔥

```

---

Si tu veux, je peux aussi :
- générer la **table des matières automatique**
- ajouter un **diagramme Mermaid**
- ou transformer ce README en **doc d’architecture officielle (PDF / Confluence)**
```
