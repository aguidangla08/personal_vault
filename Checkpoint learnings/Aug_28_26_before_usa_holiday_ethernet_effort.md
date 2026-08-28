Projecte de ethernet a Clue technologies
# Comparació amb altre gent

## Rafa
1. Molt de codi de python
2. Bones idees
3. Big picture/claredat
4. ordre idees
5. codi més simple i llarg

## Leo
1. Molta experiència
2. Treball amb varies coses a la vegada
3. Sistema gros, multiples plataformes/dispositius
4. Sap configurar servidors/serveis
5. Architectura chula
6. Treball ben acabat i am

# Reptes

## TDAH
El deficit d'atenció i  falta de memòria em causa differents reptes

1. Inflexible cognitive model (MIrar carpeta Personal)
2. Crea la tendència de centrar-me més amb els detalls que amb la cosa, el que té punts positius i negatius com la dificultat de decisió y els bucles al no saber escollir en tradeoffs, això està relacionat amb la falta d'experiència també
3. Dificulatat de claretad mental i  structura

## Tecnic
Parts tecniques complicades

1. Arquitectura
	1. La experiència es essencial
	2. molts traddeoffs
	3. Necesitat de coneixèr el criteri de la empresa/necesitat del moment i saberlo extrapolar

# Formal
La verificació formal es complexa

## Lliçons
1. El temps de computació pot ser molt llarg i difícil de predir
	1. Si una señal te molts possible estats i fa drive a una segona señal, aquesta segona señal pot ser dificíl de verificar
	2. Si 

# Meetings
En els meetings he de ser clar i agradable, aconseguir el que vull, i donar el que els altres necesiten

## Accions concretes
1. Presentacions
	1. Afegir la informació que donaria per un issue
		1. Explicar estructura de la reunió abans de la explicació
		2. Parlar més de exemples concrets i no de forma abstracta
	2. Enviar la informació abans per si es volen preparar els companys
	3. Preparar tot abans de la reunió, no afegir res nou a menys que sigui molt important
2. Conversa
	1. Deixar parlar esperar per pauses
	2. Crear pauses per deixar parlar a la gent comodament, 1-2 s
	3. Dir de tant en quant com per exemple al principi de una secció "param si no queda clar"
	4. Acceptar els bons punts que facin els demés, no veure-ho com un atac al competir per idees o tenir més feina, al final guanyem tots
	5. Al voler afegir nova informació a la reunió, criticar un punt o matisar algo, fer servir una formula similar a:
		1. "Em sembla interesant x perquè y, està molt bé perquè z, jo afegiria w"
		2. Es suau, acceptes la bona feina dels demes, dones oportunitat a que et diguin que no s'han explicat bé si cal i a la vegada afegeixes allò nou, sense un "però"

# Qualitat i rapidesa
He d'aconseguir produir un treball de qualitat y complir amb les dates de entrega

## Accions concretes
1. Pensar menys en refactors, si es ràpid d'aplicar i **se segur que la meva proposta es adecuada** fer-ho i ja està, si no és ràpid o no ho tinc clar, **obrir jira o gitlab issue** per a discutir-ho el issue hauria de tenir les seguents seccions: descripció, perquè, requisits a assolir, proposta de implementació (pasos/tasques i arquitectura) incloure exemples de codi i esquemes o representació visual
	1. Si algo que he fet em sembla **bastant bé** vol dir que no està prou bé y quan arribi la data de entrega hauré de correr, el mínim a incloure:
		1. Codi estructurat i formatejat (Només cal que sigui clar, no pasar cap linting, això es pot fer després)
		2. Noms de señals, variables, atributs... ben escollits
		3. Commentaris
		4. Principals funcionalitats que no funcionin en un PR
2. Cada entrega es una iteració en el treball, a més, el dia a dia també són iteracions, no he de voler començar per la solució més complicada, sino tenir un pla de evolució y començar amb el més sencill
3. Obrir PRs al moment, sino després he de recular en el codi, tornar a corre-ho tot, un proces molt lent

# Arquitectura
He començat a definir petites arquitectures per scripts o pel codi de verificació, però és un process complicat i que necesita de l'experiència per a tenir bon gust, criteri, qualitat, rapidesa, ...

## Accions concretes
1. Assumir que les solucions generades no serán les millors i tindran buits de decisions
2. Acumular coneixement, anar fent probes y veient quins elements son interesants
	1. Per a un script:
		1. Elements del codi
			1. tests/demos
			2. schemas
			3. models
			4. commands (optional if not a wrapper)
			5. drivers (can have another name but contains main features, in different folders)
			6. core/common (rehusable elements)
			7. configuration
				1. ymal (human readable clearly)
				2. json (standard/compact)
		2. Tecnologies
			1. typer (cli libary)
			2. pydantic (force values format library)

# Tasques
- Intentar tenir les tasques al dia

## Accions concretes
1. Si project managers estan ocupats avisar Cristina, per a obrir tasca o fer algo pel que no tinc permisos i es urgent
