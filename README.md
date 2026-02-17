# DermiTrack Project

Monorepo con git submodules.

## Setup
```bash
git clone --recurse-submodules https://github.com/Aleksei115/dermitrack-project.git

Repos
┌────────────────────────────────────────────────────────────┬───────────────────────────────┐
│                            Repo                            │          Descripción          │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ dermitrack                                                 │ App móvil React Native (Expo) │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ dermitrack-analytics                                       │ Dashboard Next.js             │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ dermitrack-supabase                                        │ Migraciones y Edge Functions  │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ READMEEOF                                                  │                               │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ git add -A                                                 │                               │
├────────────────────────────────────────────────────────────┼───────────────────────────────┤
│ git commit -m "Initial setup: monorepo with git submodules │                               │
└────────────────────────────────────────────────────────────┴───────────────────────────────┘

git branch -M main
git remote add origin https://github.com/Aleksei115/dermitrack-project.git
git push -u origin main