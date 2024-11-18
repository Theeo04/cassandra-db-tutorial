# Cassandra Database - Theory + Practical Exercise

Acest repository este impartit in 2 parti, unul de teorie(explicat cat mai simplist) si unul practic de configurare in retea al unui Cluster de CassandraDB.

## Partea de teorie

### 1. Ce este Cassandra?

Cassandra este o bază de date special concepută să gestioneze volume mari de date, să fie foarte rapidă și să poată fi scalată ușor pe mai multe servere.

### 2. Cum stochează datele Cassandra?

Nu se bazează pe relații și tabele conectate între ele, ci mai degrabă pe „grupuri de date” optimizate pentru interogări rapide.

#### 3. Cum e structurat un model de date în Cassandra?

În Cassandra, datele sunt organizate în:

- Keyspace: un fel de container, ca o bază de date, care conține tabele și setările de replicare (de câte ori sunt duplicate datele pe mai multe servere pentru siguranță).

- Tabele (Column Families): asemănătoare tabelelor din SQL, dar structurate pentru a se potrivi cu interogările specifice ale aplicației.

### 4. Chei de Partiționare și Clustering
În Cassandra, Primary Key este compusă din:

- Partition Key – determină modul în care datele sunt distribuite pe noduri.
- Clustering Key – organizează ordinea datelor într-un nod specific.

Formula:
        
        Primary Key = Partition Key + Clustering Key

1. **Partition Key:**

 - Tokenizare: Cassandra transformă `Partition Key` într-un token numeric folosind o funcție hash.
 - Token-uri pe Noduri: Fiecare nod primește un interval de token-uri, împărțind spațiul total de token-uri în cluster.
 - Atribuirea Datelor: Pentru fiecare rând nou, Cassandra calculează tokenul și îl trimite nodului corespunzător intervalului.

Astfel, Partition Key decide pe ce nod se stochează datele, asigurând o distribuție echilibrată a acestora.

2. **Clustering Key:**

După ce un rând de date a ajuns pe nodul corect, `Clustering Key` intră în acțiune pentru a organiza ordinea rândurilor în cadrul nodului respectiv.

 - Pe un nod, toate rândurile cu același `Partition Key` sunt grupate împreună.
 - Clustering Key stabilește ordinea în care aceste rânduri sunt stocate. 

De exemplu, dacă Clustering Key este o coloană de timp (timestamp), atunci datele vor fi stocate în ordine cronologică pe nod.

**Exemplu:** PRIMARY KEY (customer_id, order_date)

Fiecare valoare unică a cheii de partiție`(customer_id)` va fi stocată pe un anumit nod, în funcție de hashing-ul aplicat valorii respective. Așadar, toate înregistrările care au același `customer_id` vor fi plasate pe același nod, iar ordinea va fi data de `(order_date)`

#### Recomandări pentru Performanță:
 - Distribuție echilibrată: Alege un Partition Key care să distribuie datele uniform între noduri. Poți folosi combinații de coloane pentru a evita concentrarea datelor pe același nod.

 - Reducerea dimensiunii unui Partition Key: Încearcă să menții partițiile mici (de preferință sub 100 MB per partiție) pentru a preveni scăderea performanței citirii și scrierii.

 - Clustering Key pentru ordonare eficientă: Folosește un Clustering Key care ajută la organizarea datelor pe nod și reduce necesitatea de a face sortări la interogări.

### 5. Data Types pentru Cassandra

- **Tipuri de Date Standard** (int, float, var, bigint, etc)
- **Tipuri de Date pentru Identificatori și Date Timp:**
    1. UUID - Un identificator unic universal (128 de biți), util pentru ID-uri de rânduri.
    2. timeuuid – Similar cu UUID, dar care include și informații despre timp (util în sortarea datelor cronologic)
    3. timestamp – O valoare temporală, cu precizie la milisecundă (de obicei, format UNIX).
 
- **Tipuri de Date Colecții:**
    1. list<type> – O listă ordonată de elemente de același tip.
    2. set<type> – O mulțime de elemente unice, fără ordonare.
    3. map<key_type, value_type> – O structură de perechi cheie-valoare, similară cu un dicționar (elementele sunt unice pe baza cheilor).
 
- **Tipuri de Date Speciale:**
    1. blob – O secvență binară de date, utilă pentru stocarea datelor brute.
    2. inet – Reprezintă o adresă IP (IPv4 sau IPv6), utilă pentru stocarea IP-urilor.


### 6. Cassandra Arhitecture

**1. Replicarea**: procesul de copiere a datelor pe mai multe noduri din cluster

- Replication Factor (RF): Numărul de copii ale datelor pe care le stochezi pe noduri. De exemplu, dacă RF = 3, datele sunt copiate pe 3 noduri.
- Replication Strategy:
    1. **SimpleStrategy:** Folosit pentru un singur datacenter.
    2. **NetworkTopologyStrategy:** Folosit pentru mai multe datacentere, oferind control asupra replicării în fiecare datacenter.

*Exemplu:*

```cql
CREATE KEYSPACE my_keyspace 
WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3, 'dc2': 3};
```

Niveluri de Consistency:

- ONE: Cel puțin un nod confirmă.
- QUORUM: Majoritatea nodurilor confirmă.
- ALL: Toate nodurile confirmă.

**2. Write/Read Consistency**

`Write Consistency` se referă la numărul de noduri care trebuie să confirme o scriere pentru ca aceasta să fie considerată completă și să fie acceptată de sistem.
`Read Consistency` se referă la numărul de noduri care trebuie să confirme o citire pentru ca rezultatul să fie considerat corect și valabil.

**3. Node Gossip**

Node Gossip Protocol este un mecanism utilizat de Apache Cassandra pentru a permite nodurilor dintr-un cluster să comunice între ele și să-și sincronizeze starea.
Exemple de date transmise prin Gossip:
- Starea nodului: Activ, inaccesibil, offline, etc.
- Info despre replici: Ce date sunt stocate pe ce noduri.
- Starea memoriei: Câtă memorie folosește nodul respectiv.
- Sarcini de mentenanță: Dacă un nod este în proces de rebalansare a datelor sau alte sarcini administrative.

**4. How write works**

1. Commit Log:
    - La fiecare scriere în baza de date, datele sunt mai întâi salvate în commit log. Commit log-ul este utilizat pentru a 
        recupera datele care nu au fost încă scrise în memoria principală (Memtable) sau pe disk (SSTables) în caz de repornire sau crash al nodului.
 
 2. Memtable:
    -  După ce datele sunt scrise în commit log, ele sunt stocate în Memtable pentru o perioadă scurtă. Memtable-ul este gestionat de fiecare nod și conține datele care au fost actualizate recent. Când Memtable-ul se umple, acesta este transformat într-un fișier pe disk (SSTable) și se șterge din memorie

 3. SSTable (Sorted String Table):
    - este un fișier pe disk care conține datele stocate într-un format sortat. 
    - Este creat atunci când un Memtable este „scris” pe disk, adică datele din memorie sunt salvate pe disk.

 4. Data Files (SSTables): fiecare SSTable conține date stocate permanent și sunt organizate în blocuri de date pentru a îmbunătăți performanța citirii.

**Cum interacționează aceste componente?**

1. Datele din Memtable sunt scrise pe disk într-un fișier SSTable.
2. Commit log-ul poate fi apoi curățat pentru a economisi spațiu pe disc.
3. În momentul citirii, Cassandra va căuta mai întâi în Memtable (pentru datele cele mai recente).
4. Dacă datele nu sunt în Memtable, va căuta în fișierele SSTable.

### 7. Configuration Files

Pentru a controla comportamentul și funcționalitatea fiecărui nod din cluster.

1. `cassandra.yaml`
    - Localizare: de obicei, în /etc/cassandra/ sau în directorul de instalare al Cassandra.
    - Rol: Este cel mai important fișier de configurare, definind setările de bază ale nodului și clusterului.
    - Setări esențiale:
        - `cluster_name`: Definește numele clusterului. Toate nodurile care aparțin aceluiași cluster trebuie să aibă același cluster_name.
        - `seeds`: Adresele IP ale nodurilor seed care permit nodurilor noi să descopere clusterul.
        - `listen_address`: Adresa IP locală a nodului - de obicei adresa masinii
        - `rpc_address`: Adresa pentru conexiuni la distanță. Se setează la `0.0.0.0` pentru a permite accesul de la alte mașini.
        - `endpoint_snitch`: Specifică configurarea topologiei rețelei (e.g., GossipingPropertyFileSnitch).
        - `data_file_directories`: Locația unde Cassandra stochează datele.
        - `commitlog_directory`: Directorul unde se păstrează Commit Logs.

2. `cassandra-rackdc.properties`
    - Rol: Definește configurarea datacenterului și a rackului, folosită pentru replicare și distribuția datelor.
    - Setări:
        - dc: Numele datacenterului pentru nod (ex. dc=datacenter1).
        - rack: Numele rack-ului din care face parte nodul (ex. rack=rack1).

3. `cassandra-env.sh`:
    - Rol: Permite configurarea variabilelor de mediu pentru Cassandra.

4. `logback.xml`: controlează configurarea sistemului de log-are.

5. `jvm.options`: aici se ajustează setările de memorie, garbage collection și alte opțiuni JVM.

## Partea Practica

Crearea a 2 containere, in aceeasi retea. Acestea vor fi:

- `ubuntu-cassandra-db`: care ruleaza o imagine de Ubuntu 22.04, cu ajutorul comenzii:

        docker run -d --name cassandra-db-{1,2,3,4} --network cassandra-network-{1,2} -p 9042:9042 ubuntu:22.04 /bin/bash -c "tail -f /dev/null"

- `ubuntu-cassandra-db`: care ruleaza o imagine de Python 3.9, cu ajutorul comenzii:

        docker run -d --name connector --network cassandra-network python:3.9 /bin/bash -c "tail -f /dev/null"

Pentru a verifica rularea in aceeasi retea a celor 2 containere: `docker network inspect cassandra-network`, si vom avea

```json
"Containers": {
            "2caa30159f537744d7fd20bb30dd1009e6f939b28a5cf1cf8e423cb2df0b13d3": {
                "Name": "python-connector",
                "EndpointID": "4ca1f23898a2cb44af0b4dae4098f65e3e4f15f005fee0fd8ee079ac05341771",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16", // IP container Python
                "IPv6Address": ""
            },
            "3d1daa57bbe2b032047f41ecc815a1be4e76c94e5d8fb591ddf44a0d76bad26e": {
                "Name": "ubuntu-cassandra-db",
                "EndpointID": "df153f861b854710cbb91c717ad0e23d144217189c1d6f727e365ec694238097",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16", // IP container Cassandra
                "IPv6Address": ""
            }
        },
```

#### Instalare Cassandra pe Container-ul ce ruleaza Ubuntu

Accesam Interactive Mode cu ajutorul comenzii: `docker exec -it ubuntu-cassandra-db bash`

Rulam `docker-compose.yaml`, apoi vom avea 3 retele(2 reprezentand DC-urile):
- dc1_network
- dc2_network
- cluster_network(contine DC1 si DC2 si Python Controller-ul)

Iar fisierele de configurare a fiecarui nod de Cassandra vor fi stocate in volumele create.

Configurarea fiecarui nod se face accesand pe rand fiecare nod cu ajutorul comenzii de mai jos, iar fierele de configurare se gasesc in `/var/log/cassandra`:
```yaml        
docker exec -it <nume_container_cassandra> bash
```

Pentru a verifica statusul DB-ului vom folosi:`nodetool status` si comanda sa deschida un nou shell `cqlsh`.