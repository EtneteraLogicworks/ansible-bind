# Role bind

Role nasadí aplikaci DNS server BIND, který může být současně autoritativním DNS serverem
a rekurzivním resolverem.


## Proměnné

Konfigurace `bind` je specifikována ve slovníku `bind`.

| Proměnná            | Povinná | Výchozí                     | Popis |
| ------------------- | ------- | --------------------------- | ----- |
| recursion           | ne      |                             | Slovník nastavující rekurzivní resolver |
| .enabled            | ne      | true                        | Zapnutí řešení rekurzivních dotazů |
| .allowed_hosts      | ne      | [ 'any' ]                   | Pole povolující vzdáleným počítačům zasílat serveru rekurzivní dotazy |
|                     |         |                             |      |
| forward             | ne      |                             | Slovník nastavující přeposílání dotazů, které server nemůže rovnou vyřešit |
| .enable             | ne      | true                        | Zapnutí přeposílání dotazů |
| .servers            | ne      | [ '8.8.8.8', '8.8.4.4' ]    | Servery, na které se budou přeposílat dotazy |
| .policy             | ne      | first                       | Politika formwardingu. Možnosti jsou "first" nebo "only" viz dokumentace BIND |
|                     |         |                             |      |
| acls                | ne      |                             | Pole slovníků pro specifikaci ACL pravidel |
| .name               | ano     |                             | Jméno ACL |
| .aces               | ano     |                             | Pole adres/jmen omezující pravidlo na dané hosty |
|                     |         |                             |      |
| tsig_keys           | ne      |                             | Pole slovníků pro specifikaci TSIG klíčů |
| .name               | ano     |                             | Jméno klíče |
| .secret             | ano     |                             | Řetězec tajemství |
| .algorithm          | ne      | HMAC-SHA512                 | Algoritmus klíče |
|                     |         |                             |      |
| dnssec              | ne      |                             | Slovník pro obecnou konfiguraci DNSSEC |
| .keys_url           | ne      | viz defaults                | URL pro stažení aktuální veřejných klíčů root zón |
| .validation         | ne      | auto                        | Typ validace DNSSEC viz BINDd dokumentace |
|                     |         |                             |      |
| max_cache_size      | ne      | 100M                        | Maximální velikost RAM použitelná pro DNS cache |
| notify_source_v6    | ne      | null                        | IPv6 adresa ze které bude server posílat notifikace ostatním |
| custom_options      | ne      |                             | RAW BIND9 konfigurace, která skončí uvnitř `options` sekce |

### Nastavení zón

Ve slovníku `bind` konfigurujeme i zóny:

| Proměnná            | Povinná | Výchozí                     | Popis |
| ------------------- | ------- | --------------------------- | ----- |
| zones               | ne      |                             | Pole slovníků pro místních specifikaci zón |
| .name               | ano     |                             | FQDN zóny |
| .type               | ano     |                             | Typ zóny (`master`, `slave`, `forward`) |
| .filename           | ne      | db.{{ zone.name }}          | Jméno souboru zóny na filesystému |
|                     |         |                             |      |
| .masters            | ano*    |                             | (`slave`) Seznam master serverů |
| .slaves             | ano*    |                             | (`master`) Doporučeno pro `master` zónu. Seznam slave serverů |
| .forwarders         | ano*    |                             | (`forward`) Nutné pro `forward` zónu. Seznam name serverů pro přeposílání požadavků |
| .tsig_key           | ano*    |                             | (`master`, `slave`) Nutné pro `master` zónu, kterou budeme replikovat. Pro `slave` zónu povinné vždy  |
|                     |         |                             |      |
| .acl                | ne      | [ 'any' ]                   | Pole adres/jmen nebo existujících ACL omezující dotazy na danou zónu |
| .updaters           | ne      | [ 'none' ]                  | Pole autorizovaných hostů, které mohou aktualizovat zónu vzdáleně. Pouze pro `master` zónu |
|                     |         |                             |      |
| .nameserver         | ano*    |                             | (`master`) FQDN primárního nameserveru zóny.  |
| .email              | ne      | zone_default.email          | (`master`) E-mail administrátora zóny |
| .refresh_period     | ne      | zone_default.refresh_period | (`master`) Interval ve kterém server se slave zónou kontroluje, zda master neudělal změny |
| .retry_period       | ne      | zone_default.retry_period   | (`master`) Interval pro slave server říkající, za jak dlouho má zkusit kontaktovat master, když předchozí dotaz selhal |
| .expire_time        | ne      | zone_default.expire_time    | (`master`) Čas platnosti slave zóny. Pokud se během něj nepodaří kontaktovat master, přestane být slave zóna poskytována |
| .negative_ttl       | ne      | zone_default.negative_ttl   | (`master`) Doba cachování negativních odpovědí |
| .ttl                | ne      | zone_default.ttl            | (`master`) Výchozí TTL záznamů v zóně |
|                     |         |                             |      |
| .records            | ne      |                             | (`master`) Pole slovníků s obsahem zóny |
| ..type              | ano     |                             | Typ záznamu (A, TXT, MX, atd.) |
| ..label             | ne      | @ (<-origin)                | Label záznamu |
| ..ttl               | ne      |                             | TTL záznamu |
| ..data              | ano     |                             | Obsah záznamu |
|                     |         |                             |      |
| zone_default        | ne      |                             | (`master`) Slovník pro výchozí nastavení zón |
| ..refresh_period    | ne      | 43200                       | Výchozí refresh_period zóny |
| ..retry_period      | ne      | 180                         | Výchozí retry_period zóny |
| ..expire_time       | ne      | 1209600                     | Výchozí expire_time zóny |
| ..negative_ttl      | ne      | 1800                        | Výchozí negativní TTL zóny |
| ..ttl               | ne      | 28800                       | Výchozí TTL zón |


## Příklady použití

Konfigurace master zóny s ACL:

```yaml
acls:
  - name: 'logicworks-kancl'
    aces:
      - '172.23.0.0/24'
      - '127.23.66.0/24'
      - 'localnets'
      - 'localhost'

tsig_keys:
  - name: 'logicworks'
    secret: 'Secret=='

zones:
  - name: 'mrak.drak'
    type: 'master'
    nameserver: 'ns.foo.bar'
    slaves:
      - '172.16.243.137'
    tsig_key: 'logicworks'
    acl:
      - 'logicworks-kancl'
    records:
      - type: 'NS'
        data: 'ns.foo.bar.'
      - label: 'krak'
        type: 'A'
        ttl: 3600
        data: '192.168.239.23'
      - type: 'A'
        ttl: 3600
        data: '192.168.239.23'
```

Konfigurace slave zóny:

```yaml
tsig_keys:
  - name: 'logicworks'
    secret: 'Secret=='

zones:
  - name: 'mrak.drak'
    type: 'slave'
    tsig_key: 'logicworks'
    masters:
      - '172.16.243.130'
```

Konfigurace reverzní zóny:

```yaml
  zones:
    - name: '239.168.192.in-addr.arpa'
      type: 'master'
      nameserver: 'ns.foo.bar'
      records:
        - type: 'NS'
          data: 'ns.foo.bar.'
        - label: '23'
          type: 'PTR'
          data: 'foo.bar.'
```
