# linux-cns-reader
Script Bash per l'installazione e la configurazione di lettori smart card CNS su Linux

# Installa tutto il necessario
sudo apt install pcscd pcsc-tools libccid opensc opensc-pkcs11 libnss3-tools

# Abilita il lettore
sudo systemctl enable --now pcscd.socket

# Inserisci lettore + Tessera Sanitaria e verifica (ovresti vedere il lettore e la carta. Per uscire fai Ctrl+C.)
pcsc_scan

opensc-tool -n

CNS card

# Verifica che OpenSC veda il certificato
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --list-objects --type cert

# Per Chrome — questo è il passaggio che su Laura era stato fondamentale Questo va fatto dall'utente che utilizzerà Chrome, non da amministratore/root. Chiudi prima chrome e poi digita

rm -rf ~/.pki/nssdb
mkdir -p ~/.pki/nssdb && certutil -N --empty-password -d sql:$HOME/.pki/nssdb

# Poi registriamo OpenSC:
modutil -dbdir sql:$HOME/.pki/nssdb -add "OpenSC" -libfile /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so

# Infine controlliamo:
modutil -dbdir sql:$HOME/.pki/nssdb -list

# Lì dovremmo vedere OpenSC, il lettore e, con la TS-CNS inserita, il token della carta. Chrome su Linux utilizza il database NSS per il modulo PKCS#11, quindi questo è il pezzo necessario per arrivare alla richiesta del PIN. CAC Help. Nota importante: il comando rm -rf ~/.pki/nssdb azzera il database NSS dell'utente
