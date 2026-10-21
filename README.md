# linux-cns-reader (Linux Mint)

Guida per installare e configurare lettori smart card e Tessera Sanitaria/CNS su Linux Mint, con utilizzo della CNS anche tramite Google Chrome.

> **Importante:** i comandi relativi a Chrome/NSS devono essere eseguiti dall'utente che utilizzerà Chrome, non da `root` e non dall'account amministratore di un altro utente.

---

## PC con più utenti

Su un PC utilizzato da più persone è importante distinguere la configurazione di sistema da quella del singolo utente.

- I passi **1–2** (installazione pacchetti e attivazione `pcscd`) si eseguono **una sola volta**, utilizzando un account amministratore tramite `sudo`.
- I passi relativi a **NSS/Chrome e ai moduli PKCS#11** devono invece essere eseguiti **separatamente per ogni utente che utilizzerà la CNS con Chrome**, accedendo con quell'utente.

> Non configurare `~/.pki/nssdb` dall'account amministratore se Chrome verrà utilizzato da un altro utente.

---

## 1. Installa i componenti necessari

```bash
sudo apt install pcscd pcsc-tools libccid libacsccid1 opensc opensc-pkcs11 libnss3-tools
```

`libacsccid1` è utile in particolare per alcuni lettori ACS, ad esempio ACR38U.

---

## 2. Abilita il servizio smart card

```bash
sudo systemctl enable --now pcscd.socket
```

Se necessario:

```bash
sudo systemctl restart pcscd
```

---

## 3. Verifica lettore e Tessera Sanitaria/CNS

Inserisci il lettore e la Tessera Sanitaria/CNS.

```bash
pcsc_scan
```

Dovresti vedere il lettore, la carta inserita e il relativo ATR.

Per uscire:

```text
Ctrl+C
```

Poi verifica con OpenSC:

```bash
opensc-tool -n
```

Con una CNS compatibile dovrebbe comparire qualcosa come:

```text
CNS card
```

Questo conferma che lettore, `pcscd` e OpenSC riescono a comunicare con la carta.

### Test opzionale del PIN tramite PKCS#11

```bash
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --login --test
```

Se viene richiesta l'immissione del PIN, OpenSC riesce a raggiungere correttamente l'autenticazione della carta.

> Non è necessario completare il test se si vuole soltanto verificare che venga richiesta l'autenticazione.

---

# Configurazione per Google Chrome

Chiudi completamente Chrome prima di modificare il database NSS.

I comandi seguenti devono essere eseguiti **dall'utente che utilizzerà Chrome**.

## 4. Verifica/crea il database NSS

Chrome su Linux utilizza il database NSS dell'utente per caricare i moduli PKCS#11.

Se non esiste ancora:

```bash
mkdir -p ~/.pki/nssdb
```

```bash
certutil -N --empty-password -d sql:$HOME/.pki/nssdb
```

> Non cancellare `~/.pki/nssdb` come prima operazione: potrebbe contenere configurazioni già presenti dell'utente.

---

## 5. Registra OpenSC

Preferire il modulo `onepin`:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -add "OpenSC smartcard framework" -libfile /usr/lib/x86_64-linux-gnu/onepin-opensc-pkcs11.so
```

`modutil` può mostrare un avviso chiedendo di chiudere il browser.

Assicurarsi che Chrome sia completamente chiuso e premere **Invio** per continuare.

Controlla quindi la configurazione:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -list
```

Con la CNS inserita dovrebbero essere visibili il modulo OpenSC e, quando correttamente riconosciuti, il lettore/token.

A questo punto riavvia:

```bash
sudo systemctl restart pcscd
```

Sfila e reinserisci la CNS, apri Chrome e prova l'accesso al servizio desiderato.

---

# Se Chrome non chiede il PIN

È possibile che:

- `pcsc_scan` riconosca correttamente lettore e carta;
- `opensc-tool -n` riconosca la carta come `CNS card`;
- OpenSC riesca anche ad arrivare alla richiesta del PIN;
- ma Chrome non presenti comunque la finestra per il PIN.

Questa situazione è stata riscontrata con alcune combinazioni di CNS/lettore.

In questo caso può essere necessario utilizzare il middleware **Bit4id** oltre a OpenSC.

---

## 6. Installazione di Bit4id tramite InfoCert Sign

InfoCert Sign per Linux contiene la libreria PKCS#11 Bit4id.

Dopo aver installato InfoCert Sign, cercare la libreria:

```bash
find /opt /usr -type f -iname 'libbit4*pki.so*' 2>/dev/null
```

Con la versione di InfoCert Sign utilizzata durante i test, la libreria è stata trovata in:

```text
/opt/infocert-sign-desktop/resources/app.asar.unpacked/node_modules/@ice/dike-core-linux/native/lib/libbit4xpki.so
```

> **Importante:** il percorso può cambiare con versioni o aggiornamenti di InfoCert Sign.
>
> Utilizzare nel comando `modutil` il percorso effettivamente restituito da `find`. Il percorso riportato sopra è solamente un esempio reale riscontrato durante i test.

Chiudi completamente Chrome.

Registra quindi Bit4id nel database NSS dell'utente utilizzando il percorso trovato.

Esempio:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -add "CNS_Bit4id" -libfile /opt/infocert-sign-desktop/resources/app.asar.unpacked/node_modules/@ice/dike-core-linux/native/lib/libbit4xpki.so
```

`modutil` mostrerà un avviso simile a:

```text
WARNING: Performing this operation while the browser is running could cause
corruption of your security databases.
```

Assicurarsi che Chrome sia completamente chiuso e premere **Invio** per continuare.

Il messaggio:

```text
Module "CNS_Bit4id" added to database.
```

conferma che il modulo è stato registrato.

Controlla:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -list
```

È possibile lasciare registrati contemporaneamente:

```text
OpenSC smartcard framework
CNS_Bit4id
```

Riavvia quindi `pcscd`:

```bash
sudo systemctl restart pcscd
```

Sfila e reinserisci la CNS, quindi riapri Chrome e prova nuovamente l'accesso.

---

# Esperienza pratica

Sono state riscontrate configurazioni nelle quali **OpenSC da solo è sufficiente** e altre nelle quali Chrome non arriva alla richiesta del PIN nonostante la CNS sia correttamente riconosciuta dal sistema.

Su una postazione con:

- Linux Mint
- Google Chrome
- lettore ACS ACR38U
- Tessera Sanitaria riconosciuta da OpenSC come `CNS card`

OpenSC riconosceva correttamente lettore e carta e il test PKCS#11 arrivava alla richiesta del PIN, ma Chrome non richiedeva il PIN durante l'accesso CNS all'Agenzia delle Entrate.

L'aggiunta del modulo:

```text
CNS_Bit4id
```

tramite `libbit4xpki.so` fornita da InfoCert Sign ha risolto il problema.

**Conclusione:** non è necessario installare Bit4id preventivamente su tutte le postazioni.

Provare prima **OpenSC**.

Se la CNS viene correttamente riconosciuta dal sistema ma Chrome non arriva alla richiesta del PIN, provare **Bit4id**.

---

# Note diagnostiche

## `--list-objects` non mostra certificati

Il comando:

```bash
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --list-objects --type cert
```

può non mostrare certificati anche su una postazione dove la CNS funziona perfettamente.

**Non utilizzare quindi l'assenza di certificati nell'output di questo comando come prova che la CNS non funzioni.**

Per una diagnosi iniziale sono molto più utili:

```bash
pcsc_scan
```

```bash
opensc-tool -n
```

e, se necessario:

```bash
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --login --test
```

---

## Chrome non chiede più il PIN dopo un aggiornamento di InfoCert Sign

È possibile che un aggiornamento di InfoCert Sign modifichi il percorso della libreria Bit4id.

Se Chrome smette di chiedere il PIN, verificare nuovamente dove si trova la libreria:

```bash
find /opt /usr -type f -iname 'libbit4*pki.so*' 2>/dev/null
```

Controllare anche il modulo attualmente registrato:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -list
```

Se `CNS_Bit4id` punta a un percorso che non esiste più, chiudere completamente Chrome e rimuovere la vecchia registrazione:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -delete "CNS_Bit4id"
```

Quindi registrare nuovamente il modulo utilizzando **il nuovo percorso restituito da `find`**.

Esempio:

```bash
modutil -dbdir sql:$HOME/.pki/nssdb -add "CNS_Bit4id" -libfile /nuovo/percorso/libbit4xpki.so
```

Riavviare quindi:

```bash
sudo systemctl restart pcscd
```

Sfilare e reinserire la CNS e riaprire Chrome.

---

# Ripristino NSS — solo in caso di problemi

La cancellazione:

```bash
rm -rf ~/.pki/nssdb
```

**azzera il database NSS dell'utente.**

Non deve quindi essere utilizzata come normale passaggio di installazione.

Usarla soltanto se si vuole deliberatamente ricreare da zero la configurazione NSS dell'utente.

Dopo aver cancellato il database sarà necessario ricrearlo e registrare nuovamente i moduli PKCS#11 necessari.
