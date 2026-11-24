# TP1 SOAP et WSDL - Réponses

Binôme : 24110 et 23001

Environnement : JDK 17, Maven 3.9, jaxws-rt 4.0.2, SoapUI, curl (Git Bash).

---

## Partie A : Code-first

### Étape 1 : Créer et lancer le service

Après `mvn compile exec:java`, la console affiche `WSDL : http://localhost:8080/bib/EmpruntService?wsdl` et le WSDL s'ouvre bien dans le navigateur.

**Q1.** C'est l'annotation `@WebService` sur la classe `EmpruntService` qui en fait un service web. JAX-WS expose alors ses méthodes comme des opérations et génère le WSDL au démarrage.
Pour l'exception, c'est `@WebFault(name = "LivreIndisponible", targetNamespace = "http://supnum.mr/bib")`. Grâce à elle, l'exception apparaît dans le contrat : on retrouve un `<message name="LivreIndisponibleException">` et un `<fault>` dans l'opération `emprunterLivre`, et l'objet retourné par `getFaultInfo()` est envoyé dans le `<detail>` de la Fault.

**Q2.** JAXB doit recréer les objets Java à partir du XML reçu (et l'inverse pour la réponse). Il crée l'objet vide par réflexion, puis remplit les champs un par un en lisant les balises. Il ne peut pas deviner quel constructeur avec paramètres appeler ni dans quel ordre passer les valeurs. Sans constructeur vide, JAXB refuse la classe (erreur du type "does not have a no-arg default constructor") et le service ne démarre pas.

### Étape 2 : Lire le WSDL généré

**Q1.** L'URL du service est dans `<service>/<port>/<soap:address>` : `http://localhost:8080/bib/EmpruntService`.

Il y a deux opérations dans le portType `EmpruntService` :
- `emprunterLivre` : entrée = message `emprunterLivre` (idAdherent, isbn), sortie = message `emprunterLivreResponse` (un élément `emprunt` avec numero et dateRetour). Elle a aussi un fault `LivreIndisponibleException`.
- `rendreLivre` : entrée = message `rendreLivre` (numeroEmprunt), sortie = message `rendreLivreResponse` (penalite). Pas de fault déclaré.

**Q2.** Le binding contient `<soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>` et `<soap:body use="literal"/>`, donc c'est du document/literal (en mode wrapped, l'élément du Body porte le nom de l'opération).
C'est du SOAP 1.1 : le préfixe `soap` du binding correspond à `http://schemas.xmlsoap.org/wsdl/soap/`. Pour du SOAP 1.2, on aurait `http://schemas.xmlsoap.org/wsdl/soap12/`.

**Q3.** Dans le XSD (`?xsd=1`), `isbn` est de type `xs:string`, et `dateRetour` aussi est en `xs:string`, parce que dans la classe `Emprunt` c'est un `String`. Tous les éléments ont `minOccurs="0"`.
La règle de format de l'ISBN n'apparaît nulle part. Elle est seulement dans le `Pattern` Java, et le générateur ne peut pas la voir. D'après le contrat, n'importe quelle chaîne est donc acceptée.

**Q4.** Non. SoapUI génère :
```xml
<bib:emprunterLivre>
   <idAdherent>A-117</idAdherent>
   <isbn>978-1-234-5678-9</isbn>
</bib:emprunterLivre>
```
Seul le wrapper `emprunterLivre` a le préfixe. Le schéma généré n'a pas `elementFormDefault="qualified"`, donc les éléments locaux sont non qualifiés (sans espace de noms). Nous avons essayé de mettre `<bib:isbn>` : le paramètre arrive à `null` et on reçoit la Fault "ISBN mal formé : null".

**Q5.** Dans l'onglet Raw :
- HTTP : la première ligne `POST http://localhost:8080/bib/EmpruntService HTTP/1.1` et les en-têtes (`Content-Type: text/xml;charset=UTF-8`, `SOAPAction: ""`, `Content-Length`, `Host`...). HTTP sert de transport : il amène le message jusqu'à l'URL et renvoie un code de statut.
- XML : tout le corps du message. C'est le format des données, lisible par n'importe quel langage.
- SOAP : les balises `Envelope`, `Header`, `Body` dans l'espace de noms `http://schemas.xmlsoap.org/soap/envelope/`. C'est le protocole de messagerie : il définit l'enveloppe, les en-têtes optionnels et le format des erreurs (Fault).
- Le WSDL n'apparaît pas dans le message. C'est le contrat qui décrit le service : les opérations et les données (portType, types), la façon d'emballer les messages (binding) et l'adresse (service). SoapUI l'a utilisé pour générer la requête.

### Étape 3 : Envoyer des messages SOAP

`xml/01-emprunter.xml` envoyé avec curl :
```
HTTP/1.1 200 OK
...<ns2:emprunterLivreResponse xmlns:ns2="http://supnum.mr/bib"><emprunt><numero>E-2026-0812</numero><dateRetour>2026-12-08</dateRetour></emprunt></ns2:emprunterLivreResponse>
```

**Q1.** Avec SOAP, HTTP sert juste à transporter le message. L'opération à exécuter est indiquée dans le Body (`emprunterLivre` ou `rendreLivre`), pas par la méthode HTTP ni par l'URL : il n'y a qu'une seule URL pour toutes les opérations. Comme il faut envoyer une enveloppe XML dans le corps de la requête, on est obligé d'utiliser POST, même pour lire une information. (D'ailleurs un GET sur l'URL sans `?wsdl` renvoie seulement une page HTML d'information.)

**Q2.** Non, on a bien reçu une réponse 200. Le bloc `jeton` n'a pas l'attribut `mustUnderstand="1"`, il est donc facultatif. Le service ne connaît pas cet en-tête, alors il l'ignore et traite le Body normalement.

**Q3.** Le service répond avec une Fault :
```
HTTP/1.1 500 Internal Server Error
<faultcode>S:VersionMismatch</faultcode>
<faultstring>Couldn't create SOAP message. Expecting Envelope in namespace http://schemas.xmlsoap.org/soap/envelope/, but got http://www.w3.org/2003/05/soap-envelope</faultstring>
```
À l'étape 2 on a vu que le binding est en SOAP 1.1. Le service attend donc une enveloppe dans l'espace de noms SOAP 1.1. Quand il reçoit une enveloppe 1.2, il la rejette avec le code `VersionMismatch`, et la Fault elle-même est renvoyée en SOAP 1.1.

**Q4.** Les cinq erreurs :
1. `<soap:envelope>` en minuscule : il faut `<soap:Envelope>` (XML est sensible à la casse). Le serveur répond Fault Client "unexpected XML tag. expected: ...Envelope but found: ...envelope".
2. La balise fermante `</rendreLivre>` ne correspond pas à `<bib:rendreLivre>` : il faut `</bib:rendreLivre>`. Le XML est mal formé, et on reçoit une Fault (UnmarshalException).
3. L'espace de noms `http://supnum.mr/bibliotheque` est faux : il faut `http://supnum.mr/bib`, le targetNamespace du WSDL. Erreur reçue : "Cannot find dispatch method for {http://supnum.mr/bibliotheque}rendreLivre".
4. Le paramètre s'appelle `numeroEmprunt` (c'est le `@WebParam`) et il ne doit pas avoir de préfixe. Il faut donc `<numeroEmprunt>` au lieu de `<bib:numero>`.
5. Le `Header` est placé après le `Body` : dans SOAP, l'ordre est Envelope, puis Header (optionnel), puis Body. Il faut remonter le Header au-dessus du Body.

Le message corrigé est dans `xml/04-rendre-livre.xml` et renvoie `<penalite>0</penalite>`.

Les erreurs 4 et 5 ne déclenchent aucune erreur côté serveur :
- Pour la 4, JAXB ignore simplement l'élément qu'il ne connaît pas, et le paramètre vaut `null`. Avec la version de l'étape 3, la réponse était quand même 200 avec une pénalité de 0, donc on ne voit pas le problème. (Depuis l'étape 4, on obtient la Fault "Numéro d'emprunt inconnu : null".)
- Pour la 5, Metro ne vérifie pas l'ordre Header/Body, et comme le jeton n'est pas obligatoire il est ignoré. Le message n'est pourtant pas conforme à la norme.

Ces deux erreurs sont les plus dangereuses parce qu'elles passent sans qu'on s'en rende compte.

### Étape 4 : Provoquer et construire des Faults

**Q1.**
- Cas 1 (ISBN `978-2-10`) : `faultcode = S:Client`, `faultstring = ISBN mal formé : 978-2-10`, HTTP 500.
- Cas 2 (ISBN `978-2-10-074444-4`) : `faultcode = S:Server`, `faultstring = Livre indisponible`, HTTP 500.

En SOAP 1.1, toutes les Faults sont renvoyées avec le code HTTP 500, même quand c'est la faute du client.

**Q2.** Le `<detail>` contient :
```xml
<detail>
  <ns2:LivreIndisponible xmlns:ns2="http://supnum.mr/bib">
    <isbn>978-2-10-074444-4</isbn>
    <retourPrevu>2026-12-02</retourPrevu>
  </ns2:LivreIndisponible>
</detail>
```
C'est l'objet `LivreIndisponibleInfo` transformé en XML. Dans le WSDL, l'opération `emprunterLivre` a un `<fault>` qui renvoie au message `LivreIndisponibleException`. La part de ce message pointe vers l'élément `tns:LivreIndisponible`, dont la structure (isbn, retourPrevu) est décrite dans le XSD `?xsd=1` par le type `livreIndisponibleInfo`.
Le cas 1 n'a pas de `<detail>` parce que cette Fault est créée à la main avec `SOAPFaultException` et n'est pas déclarée dans le contrat.

**Q3.** JAX-WS a mis `Server`, parce qu'il utilise ce code par défaut pour toutes les exceptions `@WebFault`. Mais ce n'est pas vraiment une erreur du serveur : le serveur a bien fonctionné. C'est la demande du client qui ne peut pas être satisfaite, parce que le livre est déjà sorti. Nous aurions choisi `Client` (ou `Sender` en 1.2) : si le client renvoie la même requête, ça échouera pareil jusqu'au retour du livre. Avec `Server`, un client pourrait croire à une panne et réessayer pour rien.

**Q4.** Avec `mustUnderstand="1"`, le serveur refuse tout le message :
```
<faultcode>S:MustUnderstand</faultcode>
<faultstring>MustUnderstand headers:[{http://supnum.mr/bib}jeton] are not understood</faultstring>
```
L'emprunt n'est pas fait. Le bloc devient obligatoire, et comme aucun code du service ne gère `jeton`, il doit rejeter la requête.
Avec `mustUnderstand="0"`, c'est comme s'il n'y avait pas l'attribut : le jeton est ignoré et on reçoit une réponse normale (200, emprunt `E-2026-0813`).

Pour la question 4 de l'étape (rendreLivre), après le redémarrage, un numéro invalide comme `X-1999-0001` donne une Fault `S:Client` "Numéro d'emprunt inconnu : X-1999-0001", et un numéro correct renvoie toujours une pénalité de 0.

**Q5.** Notre Fault 1.2 est dans `xml/03-fault-sender-12.xml` (`xmllint --noout` ne signale aucune erreur). Les différences avec les Faults 1.1 reçues :
- l'espace de noms de l'enveloppe est `http://www.w3.org/2003/05/soap-envelope` au lieu de `http://schemas.xmlsoap.org/soap/envelope/` ;
- en 1.1 le code est un simple texte `<faultcode>S:Client</faultcode>`. En 1.2 il est structuré : `<soap:Code><soap:Value>soap:Sender</soap:Value></soap:Code>`, et on peut ajouter des `Subcode` ;
- les codes changent de nom : `Client` devient `Sender` et `Server` devient `Receiver` ;
- `<faultstring>` devient `<soap:Reason><soap:Text xml:lang="fr">`, et on peut donner le message en plusieurs langues ;
- `<detail>` devient `<soap:Detail>`, qui est qualifié par l'espace de noms SOAP ;
- en SOAP 1.2 sur HTTP, une Fault `Sender` est renvoyée avec le code 400 (et non 500 comme en 1.1), et le Content-Type est `application/soap+xml`.

---

## Partie B : Contract-first

### Étape 5 : Générer le code depuis le WSDL

Après `mvn generate-sources`, on trouve dans `target/generated-sources/wsimport/mr/supnum/bib/contrat/` : `EmpruntPortType`, `EmpruntService`, `EmpruntType`, `EmprunterLivre`, `EmprunterLivreResponse`, `LivreIndisponible`, `LivreIndisponibleFault`, `ObjectFactory` et `package-info`.

**Q1.**
- Opération : `emprunterLivre` dans le portType `EmpruntPortType`, avec en entrée `emprunterLivreRequest` et en sortie `emprunterLivreResponse`.
- Message de fault : `LivreIndisponibleFault`, dont la part `fault` est l'élément `bib:LivreIndisponible`. Dans l'opération, le fault est nommé `LivreIndisponible`.
- Action SOAP : `http://supnum.mr/bib/emprunterLivre` (dans `<soap:operation soapAction="..."/>` du binding). Il faut l'envoyer dans l'en-tête HTTP `SOAPAction`.

**Q2.**
```java
@WebMethod(action = "http://supnum.mr/bib/emprunterLivre")
@WebResult(name = "emprunt", targetNamespace = "http://supnum.mr/bib")
public EmpruntType emprunterLivre(
    @WebParam(name = "idAdherent", targetNamespace = "http://supnum.mr/bib") String idAdherent,
    @WebParam(name = "isbn", targetNamespace = "http://supnum.mr/bib") String isbn)
    throws LivreIndisponibleFault;
```
(avec aussi `@RequestWrapper` et `@ResponseWrapper`). wsimport a reconnu le style wrapped et transformé les enfants de `emprunterLivre` en paramètres de la méthode. On remarque le `targetNamespace` sur chaque paramètre.

**Q3.** `dateRetour` est de type `XMLGregorianCalendar` (avec `@XmlSchemaType(name = "date")`), parce que le WSDL déclare `xs:date`. Dans la partie A, c'était un `String` et le WSDL généré disait `xs:string`, donc rien ne garantissait qu'il s'agissait d'une date. Ici, le type date fait partie du contrat.

**Q4.** La classe générée s'appelle `LivreIndisponibleFault`. Son nom vient du nom du message du WSDL : `<wsdl:message name="LivreIndisponibleFault">`. L'objet qu'elle contient (`getFaultInfo()`) est de type `LivreIndisponible`, d'après l'élément XSD `bib:LivreIndisponible`.

### Étape 6 : Implémenter et comparer

**Q1.** Oui, le WSDL publié est bien notre fichier : mêmes préfixes (`wsdl:`, `xs:`, `bib:`), même binding `EmpruntSoapBinding`, schéma directement dans le WSDL. Metro ajoute juste un commentaire en haut et met à jour l'adresse. On y trouve bien `<xs:pattern value="97[89]-\d-\d{2,5}-\d{2,6}-\d"/>` et `type="xs:date"` pour `dateRetour` et `retourPrevu`.

**Q2.** Oui. Avec `01-emprunter.xml` tel quel, on obtient une Fault Server : `Cannot invoke "String.equals(Object)" because "isbn" is null`. Le schéma de ce WSDL a `elementFormDefault="qualified"`, donc `idAdherent` et `isbn` doivent être dans l'espace de noms du service. Il faut écrire `<bib:idAdherent>` et `<bib:isbn>`. Sans préfixe, JAXB ne les reconnaît pas et les paramètres arrivent à `null`. Après correction, on reçoit 200 avec l'emprunt `E-2026-0812`. Avec l'ISBN `978-2-10-074444-4`, on reçoit la Fault `Livre indisponible` avec le `<detail>` (isbn, retourPrevu 2026-12-02).
Nous avons aussi mis l'en-tête `SOAPAction: "http://supnum.mr/bib/emprunterLivre"` comme demandé par le contrat.

**Q3.** Sans `@SchemaValidation`, l'ISBN `978-2-10` n'est pas refusé : on reçoit un emprunt normal (200). Le pattern est dans le WSDL, mais personne ne le vérifie, et notre implémentation ne teste que l'ISBN déjà emprunté.
Avec `@SchemaValidation`, le message est refusé avant d'arriver à notre code :
```
<faultcode>S:Client</faultcode>
<faultstring>org.xml.sax.SAXParseException; cvc-pattern-valid: Value '978-2-10' is not facet-valid with respect to pattern '97[89]-\d-\d{2,5}-\d{2,6}-\d' for type 'IsbnType'.</faultstring>
```
C'est une Fault Client, ce qui est logique puisque c'est la requête qui ne respecte pas le contrat.

**Q4.** Sans `wsdlLocation`, Metro ne publie plus notre fichier et génère un nouveau WSDL à partir des classes Java (comme en partie A) :
- le schéma n'est plus dans le WSDL, il est importé avec `schemaLocation="...?xsd=1"` ;
- certains noms changent : le binding devient `EmpruntPortBinding`, le message d'entrée `emprunterLivre`, le fault `LivreIndisponibleFault` ;
- des attributs `wsam:Action` sont ajoutés ;
- surtout, le type `IsbnType` et son pattern disparaissent : `isbn` redevient un simple `xs:string`.

Conséquence : même avec `@SchemaValidation`, l'ISBN `978-2-10` est de nouveau accepté, puisque la validation se fait avec ce schéma généré. On perd donc l'intérêt du contract-first. Nous avons remis `wsdlLocation` après le test.

---

## Conclusion

**Q1.** Deux différences entre les deux WSDL :
1. Le schéma de la partie A est très pauvre : tout est en `xs:string` avec `minOccurs="0"`, sans motif pour l'ISBN, et `dateRetour` est une chaîne. Celui de la partie B a un type `IsbnType` avec un pattern, des `xs:date` et des éléments obligatoires. Le contrat B est beaucoup plus précis et on peut valider les messages avec.
2. Dans la partie A, les paramètres sont non qualifiés (`<isbn>`) ; dans la partie B, ils sont qualifiés (`<bib:isbn>`), à cause de `elementFormDefault="qualified"`. C'est pour ça que le même fichier XML ne marche pas dans les deux.

On peut aussi citer la SOAPAction (vide en A, `http://supnum.mr/bib/emprunterLivre` en B), les noms différents (portType `EmpruntService` / `EmpruntPortType`, binding `EmpruntPortBinding` / `EmpruntSoapBinding`), et le fait que le WSDL B ne contient que `emprunterLivre`.

**Q2.** C'est le projet code-first (partie A) qui voit son contrat changer, parce que le WSDL est regénéré à partir du code à chaque démarrage. En partie B, le contrat est le fichier WSDL : on ne peut pas renommer le champ sans modifier le WSDL, donc ça ne peut pas arriver par accident.
Le risque pour les clients : leur code a été généré avec l'ancien WSDL et attend `<dateRetour>`. Ils vont recevoir un élément avec un autre nom, qui sera ignoré, et la date sera `null` sans aucun message d'erreur (ou alors la désérialisation plantera chez les clients plus stricts). Un simple renommage casse le contrat sans que personne ne s'en rende compte.

**Q3.** Pour le service Paiement, nous choisirions le contract-first. Il est utilisé par trois applications dans trois langages différents (.NET, PHP, Node.js). Le contrat doit donc être défini et validé avec toutes les équipes, rester stable et ne pas dépendre de l'implémentation Java. Chaque équipe génère son client à partir du même WSDL, et les règles (formats, dates, montants) sont dans le contrat.
Pour un prototype interne de deux jours, le code-first suffit : un seul client, il faut aller vite, et le contrat change souvent. Si le prototype devient un vrai service, il faudra figer le WSDL avant de le mettre en production.
