# Status: Experimental Mode
## EnemySwingTimer : Next-swing & cast timer for healers

### Description
A standalone WoW Classic (Anniversary / TBC) addon for **healers**. It predicts the
**next enemy melee swing** and shows the **current enemy cast** of the relevant unit, so
you can time heals to land right after a boss swing or cast. Unlike single-unit tools
(e.g. target-of-target only), it resolves which unit to watch via a configurable
priority list.

### Features
- Hybrid swing estimator — seeds instantly from `UnitAttackSpeed` and continuously self-corrects from observed swing timings (no manual attack-speed tables).
- Priority-based unit resolver — automatically follows target / target-of-target / focus instead of a single locked unit, each source individually toggleable.

### Github
- https://github.com/Auraliia/enemy-swing-timer
