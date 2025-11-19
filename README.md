shiftlog-nurse-app/
├── frontend/                ← React + Vite + Tailwind + Shadcn/ui
│   ├── src/
│   │   ├── components/      ← PatientCard, SystemsChecklist, SidebarBrain
│   │   ├── pages/
│   │   │   ├── Home.tsx     ← Today’s patient list
│   │   │   ├── PatientProfile.tsx
│   │   │ └── Login.tsx
│   │   ├── lib/supabase.ts  ← Supabase client
│   │   └── App.tsx
├── supabase/                ← Database schema (migrations)
│   └── migrations/
├── .github/workflows/       ← Auto-deploy to Vercel on every push
└── README.md                ← How to run in < 3 minutes
