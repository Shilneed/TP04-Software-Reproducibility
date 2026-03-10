# TP04-Software-Reproducibility
## 1. Metadonnees du binaire
- Commande de compilation: `gcc -o hello-world hello-world.c`
- Binaire produit: `hello-world`
- Taille: `15968` octets
- Permissions: `-rwxr-xr-x`
- Architecture/type: `ELF 64-bit LSB pie executable, x86-64`, dynamiquement lie
- Fichiers intermediaires: aucun fichier intermediaire persistant n'a ete observe pour cette commande simple.

## 2. Comparaison avec les sorties d'autres etudiants
- Le code source peut etre identique, mais les binaires peuvent differer.
- Causes principales: date/heure de compilation, BuildID, version du compilateur, options de compilation, environnement systeme.

## 3. Recompiler plusieurs fois: meme output ?
- Non, pas toujours.
- Ici, le programme affiche `__DATE__` et `__TIME__` (voir `hello-world.c:4`), donc la sortie depend du moment de compilation.

## 4. Executer plusieurs fois sans recompiler: meme output ?
- Oui.
- Tant qu'on execute le meme binaire, la date/heure integrees restent identiques.

## 5. Partager le binaire avec un autre etudiant
- Le binaire peut fonctionner si l'architecture/OS/ABI sont compatibles.
- Il peut echouer sur une machine differente (ex: Linux x86_64 vers macOS ARM64) avec une erreur de format.

## 6. Partager le binaire vs partager le code source
- Bonne pratique: partager le code source.
- Raisons: portabilite, transparence, reproductibilite, possibilite de recompiler selon la machine cible.

## Reponses (Section 2.3.1)


## 1. Comparer les valeurs avec d'autres etudiants / distributions
- la sortie est: `Random number: 1804289383`.
- Sans seed explicite, `rand()` part d'un etat initial par defaut  donc le resultat est deterministe
- Entre etudiants, les valeurs peuvent etre identiques si vous avez la meme libc/plateforme.
- Sur des distributions/plateformes differentes, le resultat peut changer, car l'algorithme est non garanti identique partout.

## 2. Executions multiples sans recompiler: meme output ?
- Oui, sur ce programme la sortie est la meme a chaque execution.
- C'est donc une "randomness" reproductible, mais pas une aleatoire non deterministe.

## Reponses (Section 2.4.1)

## 1. Si on compile plusieurs fois, a-t-on toujours le meme output ? Pourquoi ?
- Pas forcement.
- La recompilation en elle-meme ne change pas la logique: la valeur depend surtout de l'instant d'execution (seed = temps courant), pas du moment de compilation.
- Test observe ici apres deux recompilations avec 1 seconde d'ecart entre executions:

- Le binaire compile est, lui, resté identique dans ce test (`sha256sum` identique avant/apres recompilation).

## 2. Si on execute plusieurs fois sans recompiler, a-t-on toujours le meme output ? Pourquoi ?
- Non, pas toujours.
- Si deux executions tombent dans la meme seconde, `time(NULL)` donne la meme seed, donc meme resultat.
- Si les executions tombent sur des secondes differentes, la seed change, donc le resultat change.


## 3. Cette version se comporte-t-elle differemment a l'execution ? Pourquoi ?
- Oui, par rapport a `RNG` (sans seed), `RNGS` introduit une seed variable basee sur l'heure systeme.
- Donc le comportement runtime est moins strictement reproductible: il depend du moment d'execution.

## Reponses (Section 2.6.1)

## 1. Implementation de l'algorithme d'approximation de pi
- Le code suit bien la methode Monte Carlo:
  - generation de points `(x,y)` uniformes dans `[0,1] x [0,1]`
  - comptage des points dans le quart de disque (`x*x + y*y <= 1`)
  - estimation `pi ~= 4 * count / n`

## 2. Effet de l'augmentation de n
- Quand `n` augment
e, l'estimation de `pi` tend a se stabiliser autour de `3.14159...`.
- Le temps d'execution augmente avec `n` (cout en `O(n)`).
- Conclusion: comportement conforme aux attentes (precision moyenne meilleure quand `n` augmente, avec un temps plus long).

## 3. Reproductibilite au build
- non reproductible au build.
- Verification faite avec deux compilations consecutives de la meme source:
  - `sha512(MonteCarlos_a) = 44e021c7...20630ea`
  - `sha512(MonteCarlos_b) = 9dd306b5...52961b`
  - checksums differents.
- Cause principale: le programme affiche `__DATE__` et `__TIME__`, ce qui injecte la date/heure de compilation dans le binaire.

## 4. Reproductibilite a l'execution
- non reproductible au runtime
- Deux executions du meme binaire donnent des sorties differentes (exemple observe):
  - run 1: `n=5377`, `pi=3.157151`
  - run 2: `n=6846`, `pi=3.145194`
- Cause: seed variable `srand(time(NULL))`, donc sequence pseudo-aleatoire differente selon le moment d'execution.

### Nouvelle implementation runtime-reproductible
- Principe:
  - seed fixe (`seed=12345`)
  - `n` default `10000`
  - pas d'horodatage d'execution
- Verif:
  - deux executions du meme binaire donnent exactement la meme sortie.

## 5. Version build-time et run-time reproductible
- Choix techniques:
  - seed fixe
  - `n` fixe
  - suppression de `__DATE__`, `__TIME__`, `time()` et `ctime()` dans la sortie
- Verif:
  - valeur SHA-512 identique sur deux compilations consecutives (meme source, meme options)
  - sortie identique sur executions successives
- Ccl : reproductible a la compilation et a l'execution dans le meme environnement.

## Codes Monte Carlo (les 3 versions)

### 1. Code initial `MonteCarlo.c`
```c
#include <stdio.h> 
 #include <stdlib.h>
 #include <time.h>

 //To compile: gcc montecarlo-pi.c -o montecarlo-pi
 //Usage: ./montecarlo-pi

int main(int argc, char* argv[]) {
double x,y,z;
 int count = 0;
 time_t started = time(NULL);
 srand(started);
 int n = (int) rand() % 10000 + 1; // Number of iterations

 printf("Compiled on %s at %s\n",__DATE__,__TIME__);
 printf("Execution started on %s",ctime(&started));

 for (int i = 0; i < n; i++) {
 x = (double) rand() / RAND_MAX;
 y = (double) rand() / RAND_MAX;
 z = x * x + y * y;
 if (z <= 1) count++;
 }

 printf("The approximation of Pi using %d iterations is %f \n", n, (count / (double) n) * 4);

 time_t stopped = time(0);
 printf("Execution stopped on %s",ctime(&stopped));

 return(0);
 }
```

### 2. Code runtime reproductible `MonteCarlo_rr.c`
```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
  const unsigned int seed = 12345u;
  int n = 10000;

  if (argc > 1) {
    int parsed = atoi(argv[1]);
    if (parsed > 0) {
      n = parsed;
    }
  }

  srand(seed);

  int count = 0;
  for (int i = 0; i < n; i++) {
    double x = (double)rand() / (double)RAND_MAX;
    double y = (double)rand() / (double)RAND_MAX;
    double z = x * x + y * y;
    if (z <= 1.0) {
      count++;
    }
  }

  printf("Compiled on %s at %s\n", __DATE__, __TIME__);
  printf("Seed=%u n=%d pi=%.10f\n", seed, n, (4.0 * (double)count) / (double)n);

  return 0;
}
```

### 3. Code build+runtime reproductible `MonteCarlo_brr.c`
```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
  const unsigned int seed = 12345u;
  const int n = 10000;

  srand(seed);

  int count = 0;
  for (int i = 0; i < n; i++) {
    double x = (double)rand() / (double)RAND_MAX;
    double y = (double)rand() / (double)RAND_MAX;
    double z = x * x + y * y;
    if (z <= 1.0) {
      count++;
    }
  }

  printf("Seed=%u n=%d pi=%.10f\n", seed, n, (4.0 * (double)count) / (double)n);

  return 0;
}
```

## Reponses (Section 3.3.1)



## 1. Combien de parametres pour etre reproductible au build et au runtime ?
- 2 parametre runtime est l'option la plus simple pour une reproductibilite stricte.
- Si un parametre est accepte, il doit etre fixe explicitement sinon la sortie peut changer.
- En pratique pour ce TP: fixer `seed` et `n`, on a le meme resultat.

## 2. Build image Listing 7 vs Listing 8 et comparer les tailles
- En general, l'image du Listing 8  est plus petite que celle du Listing 7.
- Motif: multi-stage garde seulement le binaire final et retire compilateur, headers et artefacts de build.
- Donc moins de surface d'attaque, moins de transfert

## 3. Listing 8 vs binaire de la section 2.6.1: pourquoi la difference de taille ?
- Le binaire seul ne contient que l'executable.
- Une image contient aussi des couches OCI (base image, metadata, filesystem minimal, config).
- Donc l'image est presque toujours plus grande que le binaire nu.

## 4. Role de `COPY --from=build-env` (ligne 8) : bonne ou mauvaise idee ?
- C'est une bonne idee dans ce contexte.
- Avantages: image plus petite, plus propre, moins de dependances inutiles, meilleure securite.


## 6. Avec les memes input parameters, obtient-on la meme output entre etudiants ?
- Pas garanti automatiquement.
- Oui si environnement equivalent et application deterministe.


## 7. podman save / podman loadp: qu'est-ce qui est transfere ?
- On transfere l'image OCI pas seulement le binaire.
- Cela reproduit l'environnement conteneur de maniere tres fidele.

