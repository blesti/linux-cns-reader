# linux-cns-reader (Mint)

Script Bash per l'installazione e la configurazione di lettori smart card CNS su Linux

## Installa tutto il necessario
```bash
sudo apt install pcscd pcsc-tools libccid opensc opensc-pkcs11 libnss3-tools
```

## Abilita il lettore
```bash
sudo systemctl enable --now pcscd.socket
```

## Inserisci lettore + Tessera Sanitaria e verifica (dovresti vedere il lettore e la carta. 
## Per uscire fai Ctrl+C.)
```bash
pcsc_scan
```

```bash
opensc-tool -n
```

## Dovrebbe riconoscerla come "CNS card"

## Verifica che OpenSC veda il certificato
```bash
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --list-objects --type cert
```

## Per Chrome — questo è il passaggio dall'utente che utilizzerà Chrome per utilizzo CNS, non da amministratore/root. Chiudi prima chrome e poi digita

```bash
rm -rf ~/.pki/nssdb
```
```bash
mkdir -p ~/.pki/nssdb && certutil -N --empty-password -d sql:$HOME/.pki/nssdb
```

## Poi registriamo OpenSC:
```bash
modutil -dbdir sql:$HOME/.pki/nssdb -add "OpenSC" -libfile /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
```

## Infine controlliamo:
```bash
modutil -dbdir sql:$HOME/.pki/nssdb -list
```

Lì dovremmo vedere OpenSC, il lettore e, con la TS-CNS inserita, il token della carta. Chrome su Linux utilizza il database NSS per il modulo PKCS#11, quindi questo è il pezzo necessario per arrivare alla richiesta del PIN. CAC Help. Nota importante: il comando rm -rf ~/.pki/nssdb azzera il database NSS dell'utente
