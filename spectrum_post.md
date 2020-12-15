Ieri, al lancio della fatidica versione 1 dell'hub (la 2020.12.0, per l'esattezza), sono stati introdotti i **blueprint**, una sorta di automazioni modello da richiamare per costruire diverse automazioni dello stesso tipo. L'aggiornamento porta due esempi di blueprint nella directory `<config>/blueprints/automation/homeassistant`, che può essere popolata a proprio piacimento, a partire da "/automation", mentre sul sito ufficiale c'è la miniguida alla creazione del primo blueprint (uno dei due di esempio, accensione luce al passaggio davanti a un rilevatore di movimento).

Ebbene, nel mio frontend avevo, fino a poche settimane fa, un mucchio di _conditional card_ che aprivo con un _input\_select_ o un _input\_boolean_ ciascuna, finché non sono passato, per esigenza, ad automazioni che, in base al valore scelto nel select, impostavano i relativi boolean su ON/OFF.
Questo l'ho fatto perché avevo, per esempio, la scheda con le impostazioni del termostato, quella con le teste termostatiche, quella dell'acqua calda e quella con tutti i valori, quindi con codice yaml ripetuto (1. termostato, 2. teste, 3. acqua calda, 4. termo + teste + acqua). Con un'automazione con un _choose_ impostavo dunque uno o più boolean on/off a seconda del valore di un select relativo alle schede, in modo da mostrarne una sola o tutte quante.

Per 2-3 view del frontend avevo quindi diverse automazioni con vari "case" nel choose (una per i termostati, una per tutte le schede di una view, una per pressione, temperatura e umidità o indice di Thom e indice termoigrometrico, una per la vista del programma di riscaldamento, etc.) che richiamavano un unico script che faceva una cosa banalissima: lanciava il servizio _homeassistant.turn\_off_ su un gruppo costituito dai boolean delle schede da nascondere relative al select e poi un _input\_boolean.turn\_on_ per abilitarne solo una o più di esse, oppure un _homeassistant.turn\_on_ sul gruppo stesso per mostrare tutte le schede di quella sezione.

Usciti i blueprint, e capito che diavolo fossero, ho messo subito il choose lì dentro (è un'automazione, deve prenderlo, no?) invece che nell'automazione, accorciando i case in ciascuna automazione (per una ne avevo 7, in un paio solo 4, in un'altra tipo 12) a soli 3: nascondi tutto, mostra tutto, mostra solo una scheda, usando poi i trigger per scegliere quest'ultima eventualità (il CASE (2) nel codice che segue).

In poche parole, **prima avevo X automazioni con Y casi che chiamavano uno script, adesso ho X automazioni che chiamano un blueprint con 3 soli casi**.

Ho saltato il passaggio intermedio in cui X automazioni con 3 casi avrebbero potuto chiamare lo script.
Con i _blueprint_ i casi stanno tutti in esso, riducendo di fatto il codice di ciascuna automazione.

Ecco il _blueprint_:

```
# DEFINITION

blueprint:
  name: "[system] panels & cards selector"
  description: "Selezione di pannelli e card della vista"
  domain: automation
  input:
    mode:
      name: mode
      description: selected mode
      selector:
        entity:
          domain: input_select
    entities:
      name: panels or cards
      description: panels/cards to show/hide
      selector:
        target:
          entity:
            domain: group


# AUTOMATION

mode: single

trigger:
  - platform: state
    entity_id: !input mode

action:
  - choose:
      # MODE (0): hide all (reset view)
      - conditions: "{{ is_state(trigger.entity_id, '❌ nascondi tutto') or is_state(trigger.entity_id, 'nessuna selezione') }}"
        sequence:
          - service: homeassistant.turn_off
            target: !input entities
      # MODE (1): show all
      - conditions: "{{ is_state(trigger.entity_id, '🌐 mostra tutto') }}"
        sequence:
          - service: homeassistant.turn_on
            target: !input entities
      # MODE (2): show selection
      - conditions: "{{ is_state(trigger.entity_id, trigger.to_state.state) }}"
        sequence:
          - service: homeassistant.turn_off
            target: !input entities
          - service: homeassistant.turn_on
            data:
              # entity_id: "{{ 'input_boolean.' + 'card_' + (trigger.to_state.state | replace(' ', '_')) }}"
              entity_id: >
                {% set booleans = states.input_boolean | selectattr('name', 'defined') | list %}
                {% for boolean in booleans %}
                  {{ boolean.entity_id if boolean.name == trigger.to_state.state }}
                {% endfor %}
```

Commentato, nell'ultimo blocco, c'è la vecchia versione di passaggio che non riuscivo a far funzionare e poi ho sistemato (@snakuzzo, questo stavo cercando di fare ieri 😁). La forma con il for permette di utilizzare un qualunque nome per i boolean (la precedente voleva che il boolean si chiamasse come la voce del select), purché sia definito il campo _name_ e il suo valore sia uguale a quello contenuto tra le _options_ del select, ovviamente.

Qui di seguito, invece, un esempio (direttamente da casa mia) con le definizioni a monte (_input\_boolean_ per ciascuna card, e _input\_select_ per la loro scelta, entrambi da inserire, i primi, come condition di una _conditional card_ in una vista del frontend, il secondo in un a card _type: entities_ per selezionare la scheda da mostrare:

```
# INPUTS BOOLEAN

input_boolean:

  # switches
  switches_batteries_card:
    name: pulsanti
    initial: off
    icon: mdi:battery

  # sensors
  sensors_batteries_card:
    name: sensori
    initial: off
    icon: mdi:battery


# INPUTS SELECT

input_select:

  # ZigBee sensors & switches batteries
  zigbee_batteries_cards:
    name: batterie di pulsanti e sensori ZigBee
    icon: mdi:battery
    options:
      - ❌ nascondi tutto
      - 🌐 mostra tutto
      - pulsanti
      - sensori


# GROUPS

group:

  # ZigBee batteries cards
  zigbee_batteries_cards:
    name: ZigBee batteries cards
    entities:
      - input_boolean.switches_batteries_card
      - input_boolean.sensors_batteries_card
    all: false
```

Ed ecco l’automazione di esempio che invoca il blueprint:

```
# AUTOMATIONS

automation:

  # ZigBee batteries panel
  - id: system-zigbee_batteries_card_selector
    alias: "[system] ZigBee batteries card selector"
    description: "Selezione della card delle batterie dei dispositivi ZigBee"
    initial_state: true
    use_blueprint:
      path: system/card_selector.yaml
      input:
        mode: input_select.zigbee_batteries_cards
        entities:
          entity_id: group.zigbee_batteries_cards
```

Io creo dei package con gli ultimi 2 blocchi di codice per ciascuna view del frontend che preveda una cosa del genere, mentre il blueprint risiede, in questo caso sulla mia installazione, per farvi capire, in `/home/homeassistant/.homeassistant/blueprints/automation/system`, ma nell’invocazione, come si vede, si utilizza solo l’ultima directory (`system`).

Prossimamente su questi schermi, prevedere il caso "voglio attivare X schede" invece di una sola o tutte e, se possibile e non inutile (come all’inizio mi sembrava quell che stavo facendo, ma visto di quanto ho ridotto il codice di ogni singola automazione ho capito che ho fatto bene a insistere e non darmi per vinto), convertire alcuni script che ho per Alexa.

Aggiunto link a repository su GitHub: https://github.com/SpartanWithin/card-selector/tree/main
