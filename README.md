# DNS servis i konfigurisanje DNS servisa na Windows Server 2019 (Maturski ispit — АРМ-Б8)
## Šta je ovo?
Ovo je maturski (završni) ispitni zadatak za smer **Administrator računarskih mreža**, vezan za **DNS servis** u doménskoj mreži na domenu `sss.local`. Zadatak obuhvata: dijagnostiku i otklanjanje problema sa rezolucijom imena, instalaciju i konfiguraciju **sekundarnog DNS servera**, podešavanje **zone transfera** prema specifikaciji iz priloga, i pravljenje rezervne kopije DNS konfiguracije na deljeni direktorijum sa ograničenim pravima pristupa. Za rad je potreban Windows Server 2019 (kontroler domena) i dodatna virtuelna mašina za sekundarni DNS server — najlakše u VirtualBox okruženju.

## Pitanja i odgovori — DNS (Domain Name System)

### 1. Šta je DNS i čemu služi?
**DNS** (Domain Name System) je **distribuirani, hijerarhijski sistem** koji prevodi **ljudima razumljiva imena** (npr. `biblioteka1.sss.local`, `google.com`) u **IP adrese** koje koriste uređaji za međusobnu komunikaciju, i obrnuto (IP → ime, preko *reverse lookup-a*). Bez DNS-a, korisnici bi morali da pamte IP adrese svakog servera ili sajta kome žele da pristupe, što je u praksi neizvodljivo.

---

### 2. Kakva je priča iza DNS-a — zašto je nastao?
Pre DNS-a (krajem 1970-ih, u doba ARPANET-a, prethodnika interneta), svako računar je čuvao lokalnu **`HOSTS` datoteku** u kojoj su ručno bila upisana imena i IP adrese svih ostalih računara u mreži. Tu datoteku je centralno održavao **Stanford Research Institute (SRI)**, i svi računari su je periodično preuzimali (download-ovali) sa centralnog servera.

Problem je bio **skalabilnost** — kako je broj povezanih računara rastao, ova datoteka je postajala sve veća, a njeno ažuriranje i distribucija sve teža i sporija (svaki novi računar značio je novi unos koji mora da stigne do svih ostalih). Rešenje je predložio **Paul Mockapetris** 1983. godine (RFC 882/883, kasnije RFC 1034/1035) — umesto jedne centralizovane datoteke, sistem **distribuiranih, hijerarhijski organizovanih baza podataka**, gde svaki server "zna" samo za svoj deo imenskog prostora, a upiti se prosleđuju (delegiraju) drugim serverima po potrebi. Tako je nastao DNS kakav danas poznajemo.

---

### 3. Kako je organizovan DNS imenski prostor (hijerarhija)?
DNS imenski prostor je organizovan kao **obrnuto drvo (inverted tree)**:
- **Root (korenska) zona** — predstavlja se tačkom (`.`), na vrhu hijerarhije; održava je svetski mali broj **root servera**.
- **TLD (Top-Level Domain)** — npr. `.com`, `.org`, `.rs`, `.local` (privatni, ne postoji na javnom internetu).
- **Domen druge nivoa** — npr. `sss.local`, `google.com`.
- **Poddomeni / hostovi** — npr. `biblioteka1.sss.local`.

Svaki nivo "deleguje" odgovornost za niže nivoe drugim DNS serverima, čime se postiže **decentralizacija** i otpornost na otkazivanje jednog servera.

---

### 4. Kako funkcioniše DNS rezolucija (upit za ime)?
Kada klijent zatraži rezoluciju imena, postoje dva osnovna tipa upita:
- **Rekurzivni upit (recursive query)** — klijent traži od svog DNS servera **kompletan odgovor**; ako server nema podatak u svojoj bazi/kešu, on **sam** nastavlja upit dalje (ka drugim serverima) u klijentovo ime, i vraća konačan odgovor.
- **Iterativni upit (iterative query)** — server koji prima upit, ako nema odgovor, vraća **upućivanje (referral)** na drugi server koji možda ima podatak, umesto da sam nastavlja potragu — onaj koji je postavio upit mora sam da kontaktira sledeći server.

Tipičan put rezolucije javnog imena: klijent → lokalni DNS server (rekurzivni upit) → root server → TLD server → autoritativni server za domen → odgovor se vraća unazad i **kešira** na svakom nivou radi bržih budućih upita.

---

### 5. Koji su osnovni tipovi DNS zapisa (resource records)?
| Tip zapisa | Namena |
|---|---|
| **A** | Prevodi ime u IPv4 adresu |
| **AAAA** | Prevodi ime u IPv6 adresu |
| **CNAME** | "Alias" — preusmerava jedno ime na drugo (canonical) ime |
| **PTR** | Reverse lookup — prevodi IP adresu u ime (koristi se u reverse zoni) |
| **NS** | Označava koji server je autoritativan (Name Server) za zonu |
| **SOA** | Start of Authority — sadrži administrativne podatke o zoni (serijski broj, refresh, retry, expire...) |
| **MX** | Usmerava mail saobraćaj ka mail serveru domena |

---

### 6. Šta je Forward Lookup Zone, a šta Reverse Lookup Zone?
- **Forward Lookup Zone** — standardna zona koja prevodi **ime → IP adresu** (sadrži A/AAAA zapise). Ovo je ono na šta većina ljudi mislI kada kažu "DNS".
- **Reverse Lookup Zone** — zona koja prevodi **IP adresu → ime** (sadrži PTR zapise), organizovana po specijalnom domenu `in-addr.arpa` (za IPv4). Koristi se npr. kod logovanja, dijagnostike mreže, ili kod nekih sigurnosnih provera (mail serveri često provjeravaju reverse zapis pošiljaoca).

---

### 7. Zašto je server u zadatku vraćao grešku pri pristupu po imenu, a radio je po IP adresi?
Greška `Windows cannot access \\biblioteka1.sss.local` uz uspešan pristup preko IP adrese je **klasičan simptom problema sa DNS rezolucijom imena**, ne sa samom mrežnom povezanošću (jer IP konekcija radi). Najčešći uzroci ovakvog problema su:
- Klijent (računar iz zbornice) ima u svojim TCP/IP podešavanjima **pogrešno ili nepodešeno DNS server polje** (ne pokazuje na DNS server domena), pa ne može da pita ispravan server za ime.
- DNS server **nema odgovarajući A zapis** za `biblioteka1` u zoni `sss.local` (zapis nije kreiran ili je obrisan/pogrešno upisan).
- DNS klijent servis na samom računaru ima **zastareo (kеширан) negativan odgovor** u svom lokalnom DNS kešu.

U praksi, dijagnostika počinje komandama poput:

nslookup biblioteka1.sss.local

ipconfig /all

ipconfig /flushdns

čime se provjerava koji DNS server klijent koristi i da li taj server zna za traženo ime, nakon čega se greška ispravlja podešavanjem ispravne DNS adrese na klijentu i/ili dodavanjem nedostajućeg zapisa na serveru.

---

### 8. Šta je Primary, a šta Secondary DNS server, i zašto se koristi sekundarni?
- **Primary (primarni) DNS server** — server na kome se **kreira i uređuje** zona (čita/zapisuje izvorne podatke). Sve izmene zapisa rade se prvo tu.
- **Secondary (sekundarni) DNS server** — server koji drži **read-only kopiju** zone, preuzetu sa primarnog servera putem **zone transfera**. Ne uređuje se direktno.

Sekundarni server se uvodi radi:
1. **Redundantnosti** — ako primarni server padne, sekundarni i dalje može da odgovara na DNS upite.
2. **Rasterećenja (load balancing)** — upiti se mogu raspodeliti na više servera, umesto da svi idu na jedan.
3. **Geografske distribucije** — sekundarni server može biti fizički bliže određenoj grupi korisnika, što ubrzava odgovore.

---

### 9. Šta je Zone Transfer i koje vrste postoje?
**Zone transfer** je proces kojim sekundarni DNS server **preuzima (kopira)** podatke zone sa primarnog servera. Postoje dva tipa:
- **AXFR (Full zone transfer)** — prenosi se **cela** zona, bez obzira na to koliko je podataka zaista promenjeno.
- **IXFR (Incremental zone transfer)** — prenose se **samo izmene** od poslednjeg poznatog stanja zone (efikasnije, manje opterećuje mrežu).

Ko ima dozvolu da preuzme zonu (sa kojih IP adresa/servera) definiše se u svojstvima zone na primarnom serveru, u tabu **Zone Transfers**.

---

### 10. Šta je DNS Notify mehanizam?
**DNS Notify** (RFC 1996) je mehanizam kojim primarni server **aktivno obaveštava** sekundarne servere da je došlo do promene u zoni (npr. izmenjen je neki zapis), umesto da sekundarni server čeka da sam, periodično, provjeri (preko *refresh* intervala iz SOA zapisa) da li je nešto promenjeno. Ovo značajno **ubrzava propagaciju izmena** kroz mrežu — sekundarni server odmah pokreće zone transfer čim primi notifikaciju, a ne tek nakon isteka tajmera.

---

### 11. Šta su SOA parametri i šta tačno definišu?
**SOA (Start of Authority)** zapis sadrži administrativne parametre zone:
- **Serial Number** — broj verzije zone; sekundarni server ga koristi da provjeri da li ima najnoviju verziju (ako je broj na primarnom veći, traži se transfer).
- **Refresh interval** — koliko često sekundarni server provjerava primarni radi eventualnih izmena (npr. svakih 10 minuta).
- **Retry interval** — koliko sekundarni server čeka pre ponovnog pokušaja, ako prethodni pokušaj kontakta sa primarnim serverom nije uspeo (npr. 5 minuta).
- **Expire interval** — koliko dugo sekundarni server **i dalje smatra svoje podatke validnim** ako ne može da uspostavi kontakt sa primarnim serverom; nakon ovog perioda, podaci se proglašavaju nepouzdanim i server prestaje da odgovara na upite za tu zonu (npr. 12 sati).
- **Minimum TTL (Default TTL)** — podrazumevano vreme keširanja zapisa zone kod klijenata i drugih DNS servera.

*(Ovi parametri se u zadatku direktno podešavaju kroz svojstva zone — Zone Transfers / SOA tab na primarnom serveru.)*

---

### 12. Kako se prave i obnavljaju rezervne kopije DNS konfiguracije?
DNS konfiguracija (zone, podešavanja servera) može se izvesti komandom:

dnscmd /ZoneExport sss.local sss.local.dns

ili kompletna konfiguracija servera preko:

dnscmd /ExportSettings

Za potpuniju zaštitu (uključujući Active Directory-integrisane zone), preporučuje se i **System State backup** preko Windows Server Backup-a, jer AD-integrisane DNS zone nisu čisti fajlovi na disku, već deo Active Directory baze.

Vraćanje (restore) podataka radi se kopiranjem izvezenih fajlova nazad na odgovarajuću lokaciju i ponovnim uvozom (`dnscmd /ZoneAdd ... /file`), ili, kod System State backup-a, kroz **Windows Server Backup → Recovery wizard**, biranjem opcije **System State** kao tipa oporavka.

---

### 13. Koje su komande za administraciju i proveru DNS servera?
Najkorisnije PowerShell i CLI komande:
```powershell
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName "sss.local"
Get-DnsServerZoneTransferPolicy -ZoneName "sss.local"
Resolve-DnsName biblioteka1.sss.local
```

nslookup biblioteka1.sss.local

ipconfig /flushdns

ipconfig /registerdns

- `Get-DnsServerZone` — prikazuje sve zone konfigurisane na serveru (primarne, sekundarne, tip replikacije).
- `Resolve-DnsName` / `nslookup` — testiraju rezoluciju konkretnog imena, korisno za potvrdu da je problem otklonjen.
- `ipconfig /flushdns` — briše lokalni DNS keš na klijentu (korisno kad je klijent "zapamtio" stari, pogrešan odgovor).
- `ipconfig /registerdns` — ponovo registruje ime i IP adresu klijenta na DNS serveru (korisno kod dinamičkog DNS-a).

---

### 14. Zašto je važno ograničiti pristup direktorijumu sa rezervnom kopijom DNS konfiguracije?
Rezervna kopija DNS konfiguracije sadrži **osetljive podatke o celokupnoj mrežnoj infrastrukturi** (imena servera, IP adrese, strukturu domena), pa neovlašćen pristup ovim podacima predstavlja **sigurnosni rizik** (otvara mogućnost rekonstrukcije mreže za potencijalni napad). Zbog toga se direktorijum sa backup-om:
1. **Deli (share)** samo sa potrebnim korisnicima/grupama.
2. Dodatno se ograničava i preko **NTFS dozvola** (koje se primenjuju nezavisno od share dozvola, a kombinacija ta dva nivoa određuje stvarni efektivni pristup) — po pravilu, dozvolu treba dati **samo namenskom (service) korisničkom nalogu** kreiranom isključivo za potrebe backup procedure, a ne svim administratorima ili korisnicima domena.

****
