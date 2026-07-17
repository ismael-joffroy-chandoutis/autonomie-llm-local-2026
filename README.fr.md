[English](README.md) · **Français**

# Guide matériel LLM local pour l'autonomie de calcul (mi-2026)
### Mac Studio M5 Ultra 512GB vs clusters Thunderbolt 5 vs NVIDIA vs AMD, pour faire tourner des modèles open frontière à domicile

> Un guide de décision pratique et sourcé pour faire tourner des modèles de langue frontière open-weight (DeepSeek, Qwen, GLM, Kimi, MiniMax, Mistral) localement et en privé, écrit mi-juin 2026. L'idée centrale qui décide de tout : **la bande passante mémoire, pas la puissance de calcul brute, est le goulot d'étranglement pour la génération de tokens.**

*Texte sous licence CC BY-NC-ND 4.0. Toutes les données chiffrées sont sourcées et datées ; les rumeurs sont signalées comme telles.*

---

## Résumé

Faire tourner des modèles de langue frontière en local, c'est un arbitrage entre trois variables : **capacité mémoire** (loger le modèle), **bande passante mémoire** (la vitesse), et **maturité de l'écosystème**. La plupart des guides se trompent en regardant la puissance de calcul brute (TFLOPS, cœurs GPU). Pour la génération de tokens, ce qui compte c'est la bande passante.

Conclusions principales (mi-2026) :

1. **La bande passante mémoire est le goulot d'étranglement.** Sur Apple Silicon, doubler les GB/s double les tokens par seconde générés. Les cœurs GPU servent au traitement du prompt et au training, pas à la vitesse de génération.
2. **Les modèles chinois récents sont des MoE.** Ce sont les paramètres *actifs*, pas le total, qui déterminent la vitesse. C'est ce qui rend l'inférence sur mémoire unifiée viable.
3. **Un Mac à grande mémoire unifiée gagne sur la capacité, pas sur la vitesse.** Il loge un modèle de 400 à 880GB qu'aucune carte grand public ne tient, mais il le crache lentement (15 à 50 tok/s selon le modèle).
4. **Le clustering Thunderbolt 5 donne de la capacité, pas de la vitesse.** Brancher deux machines ne double pas la vitesse d'une session. Ça double le pool mémoire.
5. **Le DGX Spark et l'AMD Strix Halo sont des boîtes 128GB à faible bande passante** (256 à 273 GB/s). Ils ne peuvent ni loger ni cracher vite les vrais flagships de 2026.
6. **Le local complète les abonnements frontière, il ne les remplace pas** sur les 20% de tâches les plus dures. Il gagne sur la confidentialité, le volume illimité, et la disponibilité.

## 1. La bande passante mémoire, reine de l'inférence

À chaque token généré, la machine doit relire les poids actifs du modèle depuis la mémoire. La vitesse de génération est donc bornée par la bande passante :

> tokens/seconde ≈ (bande passante × efficacité) / (octets des paramètres actifs lus par token)

Tableau comparé (chiffres confirmés au 18/06/2026, sauf mention) :

| Matériel | Bande passante | Mémoire utilisable | Note |
|----------|---------------:|-------------------:|------|
| Apple M5 | 153 GB/s | 32GB | plafond 7B |
| Apple M5 Pro | 307 GB/s | 64GB | sweet spot 14-30B |
| Apple M5 Max (32 cœurs) | 460 GB/s | 128GB | |
| Apple M4 Max | 546 GB/s | 128GB | |
| Apple M5 Max (40 cœurs) | 614 GB/s | 128GB | meilleur die unique 2026 |
| Apple M3 Ultra | 819 GB/s | 96GB (512GB retiré, voir §6) | |
| Apple M5 Ultra | >1000 GB/s (fuites, non sorti) | jusqu'à 512GB en design (incertain) | voir §6 |
| AMD Ryzen AI Max+ 395 (Strix Halo) | ~256 GB/s (215 mesuré) | 128GB (96GB en VRAM) | |
| NVIDIA DGX Spark (GB10) | 273 GB/s | 128GB | |
| NVIDIA RTX 6000 Pro Blackwell | 1792 GB/s | 96GB | |
| NVIDIA RTX 5090 | 1792 GB/s | 32GB | référence vitesse |

Le piège : la mémoire unifiée d'un Mac n'est pas *plus rapide*, elle est plus *grande*. Une RTX 5090 va deux fois plus vite qu'un M3 Ultra mais plafonne à 32GB. Le Mac loge un modèle géant qu'aucune carte grand public ne tient, mais le crache lentement. C'est capacité contre vitesse.

Détail qui compte : le cache de contexte (KV-cache) consomme de la mémoire en plus du poids du modèle. Sur un gros modèle à 32k tokens, il peut prendre 100 à 220GB. Sur 512GB de RAM unifiée, macOS laisse environ 70 à 75% au GPU.

## 2. MoE : ce sont les paramètres actifs qui comptent

Les modèles frontière open de 2026 sont des Mixture of Experts (MoE) : seule une fraction des paramètres est activée par token. C'est ce qui rend l'inférence sur mémoire unifiée viable. Un modèle de 1 trillion de paramètres avec 32B actifs ne relit que 32B de poids par token, pas 1T.

**Sur Apple Silicon, privilégiez les MoE.** Un modèle dense de 120B est lent car tous ses paramètres sont actifs à chaque token.

## 3. Modèles frontière open, juin 2026

| Modèle | Total / actifs | Format conseillé | Empreinte | Tient sur 512GB ? | Vitesse (~1000 GB/s) |
|--------|----------------|------------------|----------:|:-----------------:|----------------------|
| **GLM-5.2** (Z.ai, 16/06) | 744B / 40B | Unsloth UD-Q4_K_XL | ~410-450GB | oui (tendu) | ~20-28 tok/s |
| **Qwen3.5-397B-A17B** (16/02) | 397B / 17B | MLX 8-bit | ~400GB | oui | ~35-55 tok/s |
| **DeepSeek V4-Flash** (24/04) | 284B / 13B | MLX 8-bit | ~285GB | oui, large | ~45-65 tok/s |
| **DeepSeek V4-Pro** (24/04) | 1,6T / 49B | UD-Q4 | ~880GB | non en Q4 (2 nœuds) | ~18-24 tok/s |
| **Kimi K2.6** (Moonshot) | 1T / 32B | UD-Q3 / Q4 | Q3 ~420GB / Q4 ~550GB | Q3 oui, Q4 non | ~25-32 tok/s |
| **MiniMax M3** (juin) | faible actifs, multimodal | MLX 4-bit | modéré | oui | rapide |
| **Mistral Large 3** | MoE (archi DeepSeek V3) | Q4/Q8 | selon taille | oui | correct |

GLM-5.2 est en tête de l'index Artificial Analysis des modèles open au moment de la rédaction. DeepSeek V4-Pro mène le SWE-bench Verified open (~80,6%). Tous sous licences permissives (MIT pour DeepSeek V4).

## 4. Quantization : Unsloth dynamic vs MLX vs full

- **Full (BF16/FP16, 2 octets/param)** : qualité maximale, mais un modèle de 671B fait 1,3TB. Seuls les modèles moyens rentrent en pleine précision.
- **Unsloth Dynamic GGUF (UD 2.0)** : garde les couches importantes (attention, première et dernière) en haute précision, compresse le reste. Environ 95% de la qualité full à la moitié de la taille. C'est la méthode pour faire tenir les géants (GLM-5.2 744B, DeepSeek V4-Pro 1,6T, Kimi 1T) sur une seule machine.
- **MLX (4/8-bit)** : quants natifs Apple Silicon, souvent plus rapides que GGUF sur Mac. À privilégier pour les modèles moyens (Qwen3.5-397B, DeepSeek V4-Flash).

## 5. Clustering Thunderbolt 5 : capacité, pas vitesse

Mythe à corriger : "si je branche deux machines, je passe de 1000 GB/s à combien ?". Mauvaise question. **Chaque machine garde sa bande passante interne.** Ce que vous ajoutez, c'est un lien Thunderbolt 5 entre les deux :

- TB5 = 80 Gbps = **10 GB/s** (mode équilibré, préféré pour le clustering).
- Avec RDMA (macOS 26.2) : latence 5 à 9 microsecondes, transfert basse latence.
- Ce pont de 10 GB/s est environ **100x plus lent** que la mémoire interne (~1000 GB/s).

Conséquence : le clustering distribue un modèle trop gros pour une machine. Il **double la capacité** (pool mémoire), il ne double **pas la vitesse** d'une session interactive. Chiffres mesurés (EXO / MLX, Apple Silicon) :

- DeepSeek 671B : **21,1 tok/s sur 1 nœud → 27,8 tok/s sur 2 nœuds** (gain ~30%, pas x2).
- Kimi K2 : **5 tok/s en lien standard → 25 tok/s avec RDMA** (c'est le RDMA qui débloque).
- Modèles trillion-params : ~28 tok/s sur cluster.

Le clustering se justifie uniquement pour loger un modèle qui ne tient pas sur une seule machine (par exemple DeepSeek V4-Pro 1,6T en Q4), ou pour servir plusieurs utilisateurs en parallèle (le throughput batch, lui, monte). Pour une utilisation interactive solo, une seule machine bien dimensionnée bat plusieurs machines mal exploitées. Et le cluster est incrémental : on ajoute un nœud plus tard.

## 6. Le contexte 512GB et la crise DRAM

Deux faits qui pèsent sur toute décision d'achat mi-2026 :

- **Apple a retiré l'option 512GB du Mac Studio M3 Ultra en mars 2026**, à cause de la pénurie DRAM (mémoire réallouée vers la HBM des accélérateurs IA). En mai 2026, le Mac Studio actuel est plafonné à 96GB. Sources : MacRumors, Tom's Hardware, Notebookcheck.
- **Tim Cook a annoncé (WSJ, 17/06/2026) des hausses de prix "inévitables"**, qualifiant la demande mémoire d'inondation centennale. Ni montant, ni calendrier précisés.

Le Mac Studio M5 Ultra n'a pas été annoncé à la WWDC (8-12 juin 2026) et est attendu vers octobre 2026. Deux camps de fuites s'opposent sur le 512GB :

- **Camp fuites supply-chain** (Notebookcheck, TrendForce, BigGo) : silicium conçu pour 36 cœurs CPU, 84 cœurs GPU, jusqu'à 512GB, bande passante >1000 GB/s.
- **Camp réalité DRAM** (Macworld, Gurman) : Apple pourrait plafonner l'Ultra à 256GB.

Conclusion honnête : le silicium peut faire 512GB, mais l'existence commerciale de cette config, et son prix, restent incertains.

## 7. Quelle machine acheter, selon l'objectif

| Objectif | Recommandation | Pourquoi |
|----------|----------------|----------|
| **Faire tourner les géants open (744B-1,6T) en local et privé** | Mac Studio M5 Ultra 512GB (1 unité) | Seule boîte grand public qui les loge en une unité |
| **Vitesse max sur 70-235B + double usage avec vidéo/diffusion** | RTX 6000 Pro Blackwell 96GB | 1792 GB/s, CUDA mature, sert aussi le rendu |
| **DeepSeek V4-Pro 1,6T en Q4 pleine taille** | 2× Mac Studio Ultra en cluster TB5 | Le seul cas qui nécessite 1TB de pool |
| **Budget, modèles ~70B** | AMD Strix Halo 128GB | ~2000 dollars, mais bande passante faible |
| **Prototypage / fine-tuning CUDA** | NVIDIA DGX Spark | Excellent prompt processing, écosystème CUDA |
| **Inférence portable, tier moyen** | MacBook Pro M5 Max 128GB | 614 GB/s, mais plafond ~100GB |

Pourquoi le DGX Spark et l'AMD ne sont pas des machines frontier : tous deux sont des boîtes 128GB à ~256-273 GB/s. Ils ne peuvent ni loger les géants (plafond 128GB vs 410-880GB nécessaires), ni les cracher vite. Preuve : le DGX Spark sur un modèle de 120B ne génère que **38,5 tok/s**, battu par trois RTX 3090 d'occasion. Le DGX Spark reste excellent pour le développement CUDA et le fine-tuning ; ce n'est pas une machine d'inférence frontier.

**Principe directeur :** le matériel le plus efficient est le moins cher qui atteint réellement l'objectif. Une boîte à 2000 dollars qui ne peut pas loger votre modèle cible n'est pas "efficiente", elle rate la cible.

## 8. L'économie de l'autonomie de calcul

Un poste mémoire élevée à 8000-12000 dollars équivaut à plusieurs dizaines de mois d'abonnements frontière. Sur la durée, c'est amortissable. Mais soyez honnête sur le **gap de qualité** : même les meilleurs modèles open en Q4 restent sous les modèles frontière fermés sur le raisonnement complexe et le code difficile, et la quantification dégrade encore.

Le local **excelle** sur : confidentialité absolue (les données ne quittent jamais la machine), volume illimité sans compteur, disponibilité permanente, et les tâches courantes (résumés, extraction, RAG privé, code routinier).

Le local **échoue** à remplacer le frontière sur les 20% de tâches les plus dures.

**Verdict : le local complète les abonnements, il ne les remplace pas.** Gardez un abonnement frontière pour le pic de difficulté, basculez le volume et le confidentiel en local.

## 9. Réserves et incertitudes

- Le Mac Studio M5 Ultra et son option 512GB ne sont pas confirmés (rumeurs supply-chain, attendu ~octobre 2026).
- Les vitesses du M5 Ultra sont extrapolées du M5 Max mesuré, à confirmer sur benchmarks réels.
- Les prix matériels sont volatils à cause de la crise DRAM ; à re-sourcer au moment de l'achat. Ajouter les taxes locales.
- Les empreintes mémoire des modèles varient selon le format de quantization exact et la longueur de contexte.

---

## Sources

- Specs et bande passante Apple M5 Pro/Max : Apple Newsroom, AppleInsider, Notebookcheck (mars 2026).
- Retrait du 512GB sur le M3 Ultra : MacRumors (05/03/2026), Tom's Hardware, Notebookcheck, Cult of Mac.
- Hausses de prix Tim Cook : Wall Street Journal via MacRumors (17/06/2026).
- Fuites M5 Ultra : Notebookcheck, TrendForce, BigGo Finance ; analyse prudente Macworld.
- Modèles : GLM-5.2 (Artificial Analysis), DeepSeek V4 (Artificial Analysis, Morph), Qwen3.5-397B-A17B (MarkTechPost), Kimi K2.6 (Artificial Analysis), MiniMax M3 (MindStudio).
- Quantization : documentation Unsloth Dynamic GGUF ; MLX.
- Clustering Thunderbolt 5 et RDMA : AppleInsider, Jeff Geerling, Creative Strategies, Engadget (macOS 26.2).
- DGX Spark : IntuitionLabs, HotHardware (273 GB/s, 38,5 tok/s en génération sur 120B, 4699 dollars).
- AMD Ryzen AI Max+ 395 : blog officiel AMD, Notebookcheck (256 GB/s, 96GB en VRAM).

## Licence

Texte sous licence [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/). Les chiffres sont sourcés à la mi-juin 2026 et vieilliront ; vérifiez les données actuelles avant toute décision d'achat. Ceci est un guide technique indépendant, pas un conseil financier ou d'achat.

Par [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
