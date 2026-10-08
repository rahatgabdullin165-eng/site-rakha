import { answer, type HistoryItem } from "@/lib/assistant";
import type { Locale } from "@/lib/types";

export const dynamic = "force-dynamic";

export async function POST(req: Request) {
  let body: { message?: unknown; locale?: unknown; history?: unknown };
  try {
    body = await req.json();
  } catch {
    return Response.json({ error: "bad_request" }, { status: 400 });
  }
  const message = typeof body.message === "string" ? body.message : "";
  const locale: Locale = body.locale === "ru" || body.locale === "kk" ? body.locale : "en";
  const history: HistoryItem[] = Array.isArray(body.history)
    ? (body.history as unknown[])
        .filter(
          (h): h is HistoryItem =>
            !!h &&
            typeof h === "object" &&
            ((h as HistoryItem).role === "user" || (h as HistoryItem).role === "assistant") &&
            typeof (h as HistoryItem).text === "string",
        )
        .slice(-8)
    : [];

  try {
    const result = await answer(message, locale, history);
    return Response.json(result);
  } catch (err) {
    console.error("assistant error", err);
    return Response.json({ error: "failed" }, { status: 500 });
  }
}
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { ensureSeeded } from "@/db/seed";
import { developers } from "@/db/schema";
import { cleanList, cleanMultiline, cleanStr, cleanUrl, sameText, slugify } from "@/lib/utils";

export const dynamic = "force-dynamic";

export async function POST(req: Request) {
  let body: Record<string, unknown>;
  try {
    body = await req.json();
  } catch {
    return Response.json({ error: "generic" }, { status: 400 });
  }
  await ensureSeeded();

  const name = cleanStr(body.name, 80);
  const role = cleanStr(body.role, 80);
  const bio = cleanMultiline(body.bio, 800);
  let handle = cleanStr(body.handle, 30).toLowerCase().replace(/^@/, "");
  if (!handle) handle = slugify(name).slice(0, 30);

  if (!name || !role || !bio || !/^[a-z0-9_-]{3,30}$/.test(handle)) {
    return Response.json({ error: "required" }, { status: 400 });
  }

  const taken = await db
    .select({ id: developers.id })
    .from(developers)
    .where(eq(developers.handle, handle))
    .limit(1);
  if (taken.length) return Response.json({ error: "handleTaken" }, { status: 409 });

  const telegram = cleanStr(body.telegram, 40);
  const wallet = cleanStr(body.wallet, 66);

  await db.insert(developers).values({
    handle,
    name,
    role: sameText(role),
    bio: sameText(bio),
    location: cleanStr(body.location, 80),
    skills: cleanList(body.skills, 14, 28),
    github: cleanUrl(body.github),
    website: cleanUrl(body.website),
    telegram: telegram ? (telegram.startsWith("@") ? telegram : `@${telegram}`) : null,
    wallet: /^0x[0-9a-fA-F]{6,64}$/.test(wallet) ? wallet : null,
    openToWork: body.openToWork !== false,
  });

  return Response.json({ ok: true, handle });
}
import { eq, sql } from "drizzle-orm";
import { db } from "@/db";
import { ensureSeeded } from "@/db/seed";
import { developers, hackathons, registrations } from "@/db/schema";
import { computeStatus } from "@/lib/types";
import { cleanStr, isEmail } from "@/lib/utils";

export const dynamic = "force-dynamic";

export async function POST(req: Request, ctx: { params: Promise<{ slug: string }> }) {
  const { slug } = await ctx.params;
  let body: Record<string, unknown>;
  try {
    body = await req.json();
  } catch {
    return Response.json({ error: "generic" }, { status: 400 });
  }
  await ensureSeeded();

  const hs = await db.select().from(hackathons).where(eq(hackathons.slug, slug)).limit(1);
  const h = hs[0];
  if (!h) return Response.json({ error: "generic" }, { status: 404 });
  if (computeStatus(h.startsAt, h.endsAt) === "past") {
    return Response.json({ error: "closed" }, { status: 400 });
  }

  const name = cleanStr(body.name, 80);
  const email = cleanStr(body.email, 120).toLowerCase();
  if (!name || !email) return Response.json({ error: "required" }, { status: 400 });
  if (!isEmail(email)) return Response.json({ error: "email" }, { status: 400 });

  const [{ n }] = await db
    .select({ n: sql<number>`count(*)::int` })
    .from(registrations)
    .where(eq(registrations.hackathonId, h.id));
  if (Number(n) >= h.maxParticipants) return Response.json({ error: "full" }, { status: 400 });

  let developerId: number | null = null;
  const handle = cleanStr(body.handle, 30).toLowerCase().replace(/^@/, "");
  if (handle) {
    const d = await db.select({ id: developers.id }).from(developers).where(eq(developers.handle, handle)).limit(1);
    if (d.length) developerId = d[0].id;
  }

  const level = ["beginner", "middle", "pro"].includes(String(body.experience))
    ? String(body.experience)
    : "middle";

  const inserted = await db
    .insert(registrations)
    .values({
      hackathonId: h.id,
      developerId,
      name,
      email,
      teamName: cleanStr(body.teamName, 60),
      experience: level,
    })
    .onConflictDoNothing()
    .returning({ id: registrations.id });

  if (!inserted.length) return Response.json({ error: "already" }, { status: 409 });
  return Response.json({ ok: true });
}
import { db } from "@/db";
import { ensureSeeded } from "@/db/seed";
import { hackathons } from "@/db/schema";
import { cleanList, cleanMultiline, cleanStr, makeSlug, sameText } from "@/lib/utils";

export const dynamic = "force-dynamic";

export async function POST(req: Request) {
  let body: Record<string, unknown>;
  try {
    body = await req.json();
  } catch {
    return Response.json({ error: "generic" }, { status: 400 });
  }
  await ensureSeeded();

  const title = cleanStr(body.title, 90);
  const description = cleanMultiline(body.description, 3000);
  const organizer = cleanStr(body.organizer, 80);
  const format = ["online", "offline", "hybrid"].includes(String(body.format))
    ? String(body.format)
    : "online";
  const startsAt = new Date(String(body.startsAt ?? ""));
  const endsAt = new Date(String(body.endsAt ?? ""));

  if (!title || !description || !organizer || isNaN(startsAt.getTime()) || isNaN(endsAt.getTime())) {
    return Response.json({ error: "required" }, { status: 400 });
  }
  if (endsAt <= startsAt) return Response.json({ error: "dates" }, { status: 400 });

  const max = Math.min(Math.max(Math.floor(Number(body.maxParticipants)) || 100, 2), 100000);
  const slug = makeSlug(title, "event");

  await db.insert(hackathons).values({
    slug,
    title: sameText(title),
    description: sameText(description),
    organizer,
    format,
    location: cleanStr(body.location, 100) || (format === "online" ? "Online" : ""),
    startsAt,
    endsAt,
    prizePool: cleanStr(body.prizePool, 60),
    maxParticipants: max,
    tags: cleanList(body.tags, 6, 24),
  });

  return Response.json({ ok: true, slug });
}
import { db } from "@/db";
import { sql } from "drizzle-orm";

export const dynamic = "force-dynamic";

export async function GET() {
  try {
    await db.execute(sql`select 1`);
    return Response.json({ ok: true });
  } catch {
    return Response.json({ ok: false }, { status: 500 });
  }
}
import { getAchievementByHash } from "@/lib/data";

export const dynamic = "force-dynamic";

export async function GET(req: Request) {
  const hash = new URL(req.url).searchParams.get("hash")?.trim() ?? "";
  if (!/^0x[0-9a-fA-F]{8,80}$/.test(hash)) {
    return Response.json({ found: false });
  }
  const credential = await getAchievementByHash(hash);
  if (!credential) return Response.json({ found: false });
  return Response.json({ found: true, credential });
}
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";
import {
  ArrowLeft,
  Award,
  BadgeCheck,
  CalendarDays,
  Code2,
  Globe,
  MapPin,
  Medal,
  Rocket,
  Send,
  Trophy,
  Wallet,
} from "lucide-react";
import { Avatar } from "@/components/Avatar";
import { CopyButton } from "@/components/actions";
import { ProjectCard } from "@/components/cards";
import { Counter, Reveal } from "@/components/fx";
import { getAchievements, getDeveloperByHandle, getProjectsByOwner } from "@/lib/data";
import { getT } from "@/lib/server-locale";
import { pickText, type AchievementKind } from "@/lib/types";
import { formatDate, shortHash } from "@/lib/ui";

export const dynamic = "force-dynamic";

const KIND_ICON: Record<AchievementKind, typeof Trophy> = {
  win: Trophy,
  finalist: Medal,
  participation: CalendarDays,
  project: Rocket,
  certificate: Award,
};

const KIND_STYLE: Record<AchievementKind, string> = {
  win: "from-gold-300 to-gold-500 text-navy-800",
  finalist: "from-sky-300 to-indigo-400 text-navy-900",
  participation: "from-slate-300 to-slate-400 text-navy-900",
  project: "from-emerald-300 to-teal-400 text-navy-900",
  certificate: "from-fuchsia-300 to-purple-400 text-navy-900",
};

export async function generateMetadata({ params }: { params: Promise<{ handle: string }> }): Promise<Metadata> {
  const { handle } = await params;
  const { locale } = await getT();
  const d = await getDeveloperByHandle(handle);
  if (!d) return { title: "Not found" };
  return { title: `${d.name} — ${pickText(d.role, locale)}`, description: pickText(d.bio, locale).slice(0, 160) };
}

export default async function DeveloperPage({ params }: { params: Promise<{ handle: string }> }) {
  const { handle } = await params;
  const { t, locale } = await getT();
  const d = await getDeveloperByHandle(handle);
  if (!d) notFound();

  const [achievements, projects] = await Promise.all([getAchievements(d.id), getProjectsByOwner(d.id)]);

  const links = [
    d.github && { icon: Code2, label: "GitHub", href: d.github },
    d.website && { icon: Globe, label: new URL(d.website).hostname, href: d.website },
    d.telegram && { icon: Send, label: d.telegram, href: `https://t.me/${d.telegram.replace(/^@/, "")}` },
  ].filter(Boolean) as { icon: typeof Code2; label: string; href: string }[];

  const placeLabel = (place: number | null) =>
    place ? (place <= 3 ? t(`place.${place}` as "place.1") : t("place.n", { n: place })) : "";

  return (
    <>
      <section className="relative overflow-hidden border-b border-white/8">
        <div className="anim-float-slow pointer-events-none absolute -right-20 -top-20 h-96 w-96 rounded-full bg-gold-400/15 blur-3xl" />
        <div className="anim-float-slow pointer-events-none absolute -left-20 top-20 h-72 w-72 rounded-full bg-sky-500/15 blur-3xl" style={{ animationDelay: "-5s" }} />
        <div className="relative mx-auto max-w-7xl px-4 pb-12 pt-10 sm:px-6 lg:px-8">
          <Link href="/developers" className="mb-6 inline-flex items-center gap-2 text-sm text-slate-300 transition hover:text-gold-300">
            <ArrowLeft size={16} /> {t("common.back")}
          </Link>
          <div className="flex flex-col gap-8 md:flex-row md:items-center">
            <div className="anim-rise relative">
              <Avatar name={d.name} seed={d.handle} size={132} className="!rounded-[2rem] shadow-[0_20px_60px_-15px_rgba(255,212,62,0.5)]" />
              {d.openToWork && (
                <span className="absolute -bottom-3 left-1/2 -translate-x-1/2 whitespace-nowrap chip chip-live">{t("common.openToWork")}</span>
              )}
            </div>
            <div className="min-w-0 flex-1">
              <h1 className="anim-rise font-display text-4xl font-extrabold tracking-tight text-white sm:text-5xl" style={{ animationDelay: "60ms" }}>
                {d.name}
              </h1>
              <p className="anim-rise mt-2 text-xl font-semibold text-gold-300" style={{ animationDelay: "120ms" }}>
                {pickText(d.role, locale)}
              </p>
              <div className="anim-rise mt-3 flex flex-wrap items-center gap-x-5 gap-y-2 text-sm text-slate-300" style={{ animationDelay: "180ms" }}>
                <span className="font-mono text-slate-400">@{d.handle}</span>
                {d.location && (
                  <span className="flex items-center gap-1.5">
                    <MapPin size={15} className="text-gold-400" /> {d.location}
                  </span>
                )}
                <span className="flex items-center gap-1.5">
                  <CalendarDays size={15} className="text-gold-400" /> {t("common.joined")} {formatDate(d.joinedAt, locale)}
                </span>
              </div>
            </div>
            <div className="anim-rise grid grid-cols-2 gap-3 sm:grid-cols-4 md:w-[430px] md:grid-cols-2" style={{ animationDelay: "240ms" }}>
              {[
                { v: d.reputation, l: t("dev.score"), gold: true },
                { v: d.winsCount, l: t("dev.wins") },
                { v: d.achievementsCount, l: t("dev.verifiedCreds") },
                { v: d.projectsCount, l: t("dev.projectsTitle") },
              ].map((s) => (
                <div key={s.l} className="card px-4 py-4 text-center">
                  <p className={`font-display text-3xl font-extrabold ${s.gold ? "text-gold-400" : "text-white"}`}>
                    <Counter value={s.v} />
                  </p>
                  <p className="mt-1 text-[11px] uppercase tracking-wide text-slate-400">{s.l}</p>
                </div>
              ))}
            </div>
          </div>
        </div>
      </section>

      <div className="mx-auto grid max-w-7xl gap-10 px-4 py-12 sm:px-6 lg:grid-cols-[1fr_340px] lg:px-8">
        <div className="space-y-12">
          <Reveal>
            <h2 className="mb-4 font-display text-2xl font-bold text-white">{t("dev.about")}</h2>
            <p className="prose-hc whitespace-pre-line text-lg">{pickText(d.bio, locale)}</p>
          </Reveal>

          <Reveal>
            <h2 className="mb-6 flex items-center gap-3 font-display text-2xl font-bold text-white">
              <BadgeCheck className="text-emerald-300" /> {t("dev.timeline")}
            </h2>
            {achievements.length === 0 ? (
              <p className="card p-6 text-slate-400">{t("dev.noAchievements")}</p>
            ) : (
              <ol className="relative space-y-5 border-l border-white/12 pl-8">
                {achievements.map((a, i) => {
                  const Icon = KIND_ICON[a.kind] ?? Award;
                  return (
                    <li key={a.id} className="anim-rise relative" style={{ animationDelay: `${Math.min(i, 8) * 70}ms` }}>
                      <span className={`absolute -left-[3.15rem] top-3 grid h-10 w-10 place-items-center rounded-xl bg-gradient-to-br ${KIND_STYLE[a.kind]} shadow-lg`}>
                        <Icon size={20} />
                      </span>
                      <div className="card card-hover spotlight p-5">
                        <div className="flex flex-wrap items-center gap-2">
                          <span className="chip">{t(`kind.${a.kind}` as "kind.win")}</span>
                          {a.place && <span className="chip chip-gold">{placeLabel(a.place)}</span>}
                          {a.verified && (
                            <span className="chip border-emerald-300/40 bg-emerald-400/15 text-emerald-200">
                              <BadgeCheck size={12} /> {t("common.verified")}
                            </span>
                          )}
                          <span className="ml-auto text-xs text-slate-400">{formatDate(a.date, locale)}</span>
                        </div>
                        <h3 className="mt-3 font-display text-lg font-bold text-white">{pickText(a.title, locale)}</h3>
                        <p className="mt-1 text-sm text-slate-400">{t("dev.verifiedBy", { issuer: a.issuer })}</p>
                        <div className="mt-3 flex flex-wrap items-center gap-2">
                          <Link
                            href={`/verify?hash=${a.txHash}`}
                            className="inline-flex items-center gap-2 rounded-lg bg-navy-950/70 px-3 py-1.5 font-mono text-xs text-sky-200 transition hover:text-gold-300"
                          >
                            ⛓ {shortHash(a.txHash)}
                          </Link>
                          <CopyButton text={a.txHash} className="!px-3 !py-1.5 !text-xs" />
                          {a.hackathonSlug && (
                            <Link href={`/hackathons/${a.hackathonSlug}`} className="text-xs font-semibold text-gold-400 hover:underline">
                              <Trophy size={12} className="mr-1 inline" />
                              {t("nav.hackathons")} →
                            </Link>
                          )}
                        </div>
                      </div>
                    </li>
                  );
                })}
              </ol>
            )}
          </Reveal>

          <Reveal>
            <h2 className="mb-6 font-display text-2xl font-bold text-white">{t("dev.projectsTitle")}</h2>
            {projects.length === 0 ? (
              <div className="card flex flex-wrap items-center justify-between gap-4 p-6">
                <p className="text-slate-400">{t("dev.noProjects")}</p>
                <Link href="/projects/new" className="btn btn-primary btn-sm">
                  {t("form.publish")}
                </Link>
              </div>
            ) : (
              <div className="grid gap-6 sm:grid-cols-2">
                {projects.map((p) => (
                  <ProjectCard key={p.slug} p={p} />
                ))}
              </div>
            )}
          </Reveal>
        </div>

        <aside className="space-y-5 lg:sticky lg:top-24 lg:self-start">
          <div className="card p-5">
            <h3 className="mb-4 font-display text-lg font-bold text-white">{t("common.skills")}</h3>
            <div className="flex flex-wrap gap-2">
              {d.skills.length ? d.skills.map((s) => <span key={s} className="chip chip-gold">{s}</span>) : <span className="text-sm text-slate-400">—</span>}
            </div>
          </div>
          {(links.length > 0 || d.wallet) && (
            <div className="card p-5">
              <h3 className="mb-4 font-display text-lg font-bold text-white">{t("dev.contact")}</h3>
              <div className="space-y-2.5">
                {links.map((l) => (
                  <a key={l.href} href={l.href} target="_blank" rel="noopener noreferrer" className="flex items-center gap-3 rounded-xl border border-white/10 bg-white/5 px-3 py-2.5 text-sm text-slate-200 transition hover:border-gold-400/50 hover:text-gold-300">
                    <l.icon size={18} className="text-gold-400" /> <span className="truncate">{l.label}</span>
                  </a>
                ))}
                {d.wallet && (
                  <div className="rounded-xl border border-white/10 bg-white/5 px-3 py-2.5">
                    <p className="mb-1 flex items-center gap-2 text-xs text-slate-400">
                      <Wallet size={14} className="text-gold-400" /> {t("dev.wallet")}
                    </p>
                    <p className="break-all font-mono text-xs text-sky-200">{shortHash(d.wallet, 10, 8)}</p>
                    <CopyButton text={d.wallet} className="mt-2 !px-3 !py-1.5 !text-xs" />
                  </div>
                )}
              </div>
            </div>
          )}
        </aside>
      </div>
    </>
  );
}
import type { Metadata } from "next";
import { ProfileForm } from "@/components/forms";
import { Container } from "@/components/PageHeader";
import { getT } from "@/lib/server-locale";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("nav.createProfile") };
}

export default function NewDeveloperPage() {
  return (
    <Container className="py-12">
      <ProfileForm />
    </Container>
  );
}
import type { Metadata } from "next";
import { DevelopersExplorer } from "@/components/Explorers";
import { Container, PageHeader } from "@/components/PageHeader";
import { getDevelopers } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("nav.developers"), description: t("page.devsSub") };
}

export default async function DevelopersPage() {
  const { t } = await getT();
  const developers = await getDevelopers();
  return (
    <>
      <PageHeader eyebrow={t("nav.developers")} title={t("page.devsTitle")} sub={t("page.devsSub")} />
      <Container>
        <DevelopersExplorer developers={developers} />
      </Container>
    </>
  );
}
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";
import { ArrowLeft, CalendarDays, Globe2, MapPin, Trophy, UserRound, Users } from "lucide-react";
import { Avatar } from "@/components/Avatar";
import { CopyButton } from "@/components/actions";
import { StatusChip, ProjectCard } from "@/components/cards";
import { RegisterForm } from "@/components/forms";
import { Reveal } from "@/components/fx";
import { getHackathonBySlug, getParticipants, getProjectsByHackathon } from "@/lib/data";
import { getT } from "@/lib/server-locale";
import { pickText } from "@/lib/types";
import { coverFor, formatDate } from "@/lib/ui";

export const dynamic = "force-dynamic";

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params;
  const { locale } = await getT();
  const h = await getHackathonBySlug(slug);
  if (!h) return { title: "Not found" };
  return { title: pickText(h.title, locale), description: pickText(h.description, locale).slice(0, 160) };
}

export default async function HackathonPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const { t, locale } = await getT();
  const h = await getHackathonBySlug(slug);
  if (!h) notFound();

  const [participants, projects] = await Promise.all([getParticipants(h.id), getProjectsByHackathon(slug)]);
  const left = Math.max(h.maxParticipants - h.participants, 0);
  const pct = Math.min(100, Math.round((h.participants / Math.max(h.maxParticipants, 1)) * 100));

  const info = [
    { icon: CalendarDays, label: t("hack.when"), value: `${formatDate(h.startsAt, locale, true)} → ${formatDate(h.endsAt, locale, true)} (UTC)` },
    { icon: MapPin, label: t("hack.where"), value: `${t(`common.${h.format}` as "common.online")}${h.location ? " · " + h.location : ""}` },
    { icon: Trophy, label: t("common.prize"), value: h.prizePool || "—" },
    { icon: Globe2, label: t("hack.organizedBy"), value: h.organizer },
  ];

  return (
    <>
      <section className="relative overflow-hidden border-b border-white/8" style={{ background: coverFor(h.slug) }}>
        <div className="absolute inset-0 bg-gradient-to-b from-navy-900/30 via-navy-900/50 to-navy-900" />
        <Trophy className="anim-float-slow absolute -right-10 top-4 h-72 w-72 rotate-12 text-white/[0.06]" />
        <div className="relative mx-auto max-w-7xl px-4 pb-12 pt-10 sm:px-6 lg:px-8">
          <Link href="/hackathons" className="mb-6 inline-flex items-center gap-2 text-sm text-slate-300 transition hover:text-gold-300">
            <ArrowLeft size={16} /> {t("common.back")}
          </Link>
          <div className="anim-rise flex flex-wrap items-center gap-3">
            <StatusChip status={h.status} />
            {h.tags.map((tag) => (
              <span key={tag} className="chip">{tag}</span>
            ))}
          </div>
          <h1 className="anim-rise mt-4 max-w-4xl font-display text-4xl font-extrabold tracking-tight text-white sm:text-6xl" style={{ animationDelay: "80ms" }}>
            {pickText(h.title, locale)}
          </h1>
          <p className="anim-rise mt-3 text-slate-200" style={{ animationDelay: "140ms" }}>
            {t("common.by")} {h.organizer}
          </p>
        </div>
      </section>

      <div className="mx-auto grid max-w-7xl gap-10 px-4 py-12 sm:px-6 lg:grid-cols-[1fr_380px] lg:px-8">
        <div className="space-y-10">
          <Reveal>
            <div className="grid gap-4 sm:grid-cols-2">
              {info.map((i) => (
                <div key={i.label} className="card flex items-start gap-4 p-5">
                  <span className="grid h-11 w-11 shrink-0 place-items-center rounded-xl bg-gold-400/15 text-gold-400">
                    <i.icon size={21} />
                  </span>
                  <div className="min-w-0">
                    <p className="text-xs uppercase tracking-wider text-slate-400">{i.label}</p>
                    <p className="mt-1 break-words font-semibold text-white">{i.value}</p>
                  </div>
                </div>
              ))}
            </div>
          </Reveal>

          <Reveal>
            <h2 className="mb-4 font-display text-2xl font-bold text-white">{t("hack.about")}</h2>
            <div className="prose-hc whitespace-pre-line text-slate-300">{pickText(h.description, locale)}</div>
            <div className="mt-4">
              <CopyButton text={`/hackathons/${h.slug}`} label={t("hack.shareLink")} />
            </div>
          </Reveal>

          {projects.length > 0 && (
            <Reveal>
              <h2 className="mb-5 font-display text-2xl font-bold text-white">{t("hack.projectsTitle")}</h2>
              <div className="grid gap-6 sm:grid-cols-2">
                {projects.map((p) => (
                  <ProjectCard key={p.slug} p={p} />
                ))}
              </div>
            </Reveal>
          )}

          <Reveal>
            <h2 className="mb-5 flex items-center gap-3 font-display text-2xl font-bold text-white">
              {t("hack.participantsTitle")}
              <span className="chip chip-gold">{h.participants}</span>
            </h2>
            {participants.length === 0 ? (
              <p className="text-slate-400">{t("hack.noParticipants")}</p>
            ) : (
              <div className="grid gap-3 sm:grid-cols-2">
                {participants.map((p) => {
                  const inner = (
                    <>
                      <Avatar name={p.name} seed={p.handle ?? p.name} size={40} className="!rounded-xl" />
                      <div className="min-w-0">
                        <p className="truncate font-semibold text-white">{p.name}</p>
                        <p className="truncate text-xs text-slate-400">
                          {p.teamName ? `${p.teamName} · ` : ""}
                          {t(`reg.${p.experience}` as "reg.middle")}
                        </p>
                      </div>
                    </>
                  );
                  return p.handle ? (
                    <Link key={p.id} href={`/developers/${p.handle}`} className="card card-hover flex items-center gap-3 p-3">
                      {inner}
                    </Link>
                  ) : (
                    <div key={p.id} className="card flex items-center gap-3 p-3">
                      {inner}
                    </div>
                  );
                })}
              </div>
            )}
          </Reveal>
        </div>

        <aside className="lg:sticky lg:top-24 lg:self-start">
          <div className="card anim-rise p-6">
            <h3 className="mb-1 flex items-center gap-2 font-display text-xl font-bold text-white">
              <UserRound size={20} className="text-gold-400" /> {t("reg.title")}
            </h3>
            <div className="mb-5 mt-4">
              <div className="mb-1.5 flex items-center justify-between text-xs text-slate-300">
                <span className="flex items-center gap-1.5">
                  <Users size={13} /> {t("common.registered", { n: h.participants, max: h.maxParticipants })}
                </span>
                {h.status !== "past" && left > 0 && <span className="text-gold-300">{t("hack.spotsLeft", { n: left })}</span>}
              </div>
              <div className="h-2 overflow-hidden rounded-full bg-white/10">
                <div className="h-full rounded-full bg-gradient-to-r from-gold-500 to-gold-300" style={{ width: `${pct}%` }} />
              </div>
            </div>
            <RegisterForm slug={h.slug} status={h.status} participants={h.participants} max={h.maxParticipants} />
          </div>
        </aside>
      </div>
    </>
  );
}
import type { Metadata } from "next";
import { HackathonForm } from "@/components/forms";
import { Container } from "@/components/PageHeader";
import { getT } from "@/lib/server-locale";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("nav.hostTournament") };
}

export default function NewHackathonPage() {
  return (
    <Container className="py-12">
      <HackathonForm />
    </Container>
  );
}
import type { Metadata } from "next";
import { HackathonsExplorer } from "@/components/Explorers";
import { Container, PageHeader } from "@/components/PageHeader";
import { getHackathons } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("nav.hackathons"), description: t("page.hackathonsSub") };
}

export default async function HackathonsPage() {
  const { t } = await getT();
  const hackathons = await getHackathons();
  return (
    <>
      <PageHeader eyebrow={t("nav.hackathons")} title={t("page.hackathonsTitle")} sub={t("page.hackathonsSub")} />
      <Container>
        <HackathonsExplorer hackathons={hackathons} />
      </Container>
    </>
  );
}
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";
import { ArrowLeft, BadgeCheck, ExternalLink, Code2, Trophy } from "lucide-react";
import { Avatar } from "@/components/Avatar";
import { CopyButton, LikeButton } from "@/components/actions";
import { ProjectCard } from "@/components/cards";
import { Reveal } from "@/components/fx";
import { getProjectBySlug, getProjects } from "@/lib/data";
import { getT } from "@/lib/server-locale";
import { pickText } from "@/lib/types";
import { CATEGORY_ICON, coverFor, formatDate, shortHash } from "@/lib/ui";

export const dynamic = "force-dynamic";

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params;
  const { locale } = await getT();
  const p = await getProjectBySlug(slug);
  if (!p) return { title: "Not found" };
  return { title: pickText(p.title, locale), description: pickText(p.summary, locale) };
}

export default async function ProjectPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const { t, locale } = await getT();
  const p = await getProjectBySlug(slug);
  if (!p) notFound();

  const all = await getProjects();
  const related = all.filter((x) => x.slug !== p.slug && (x.category === p.category || x.scale === p.scale)).slice(0, 3);

  return (
    <>
      <section className="relative overflow-hidden border-b border-white/8" style={{ background: coverFor(p.slug + p.category) }}>
        <div className="absolute inset-0 bg-gradient-to-b from-navy-900/20 via-navy-900/55 to-navy-900" />
        <div className="absolute inset-0 opacity-[0.1]" style={{ backgroundImage: "linear-gradient(#fff 1px, transparent 1px), linear-gradient(90deg, #fff 1px, transparent 1px)", backgroundSize: "32px 32px" }} />
        <span className="anim-float absolute right-[8%] top-10 hidden text-[9rem] opacity-80 drop-shadow-2xl sm:block">{CATEGORY_ICON[p.category] ?? "💡"}</span>
        <div className="relative mx-auto max-w-7xl px-4 pb-12 pt-10 sm:px-6 lg:px-8">
          <Link href="/projects" className="mb-6 inline-flex items-center gap-2 text-sm text-slate-300 transition hover:text-gold-300">
            <ArrowLeft size={16} /> {t("common.back")}
          </Link>
          <div className="anim-rise flex flex-wrap items-center gap-2">
            <span className={`chip ${p.scale === "large" ? "chip-gold" : ""}`}>{p.scale === "large" ? t("common.large") : t("common.small")}</span>
            <span className="chip">{CATEGORY_ICON[p.category]} {t(`cat.${p.category}` as "cat.web")}</span>
            {p.verified && (
              <span className="chip border-emerald-300/40 bg-emerald-400/15 text-emerald-200">
                <BadgeCheck size={12} /> {t("common.verified")}
              </span>
            )}
          </div>
          <h1 className="anim-rise mt-4 max-w-3xl font-display text-4xl font-extrabold tracking-tight text-white sm:text-6xl" style={{ animationDelay: "80ms" }}>
            {pickText(p.title, locale)}
          </h1>
          <p className="anim-rise mt-4 max-w-2xl text-lg text-slate-200" style={{ animationDelay: "140ms" }}>
            {pickText(p.summary, locale)}
          </p>
          <div className="anim-rise mt-7 flex flex-wrap gap-3" style={{ animationDelay: "200ms" }}>
            <LikeButton slug={p.slug} initialLikes={p.likes} />
            {p.demoUrl && (
              <a href={p.demoUrl} target="_blank" rel="noopener noreferrer" className="btn btn-ghost">
                <ExternalLink size={17} /> {t("common.demo")}
              </a>
            )}
            {p.repoUrl && (
              <a href={p.repoUrl} target="_blank" rel="noopener noreferrer" className="btn btn-ghost">
                <Code2 size={17} /> {t("common.repo")}
              </a>
            )}
          </div>
        </div>
      </section>

      <div className="mx-auto grid max-w-7xl gap-10 px-4 py-12 sm:px-6 lg:grid-cols-[1fr_340px] lg:px-8">
        <div className="space-y-10">
          <Reveal>
            <h2 className="mb-4 font-display text-2xl font-bold text-white">{t("project.about")}</h2>
            <div className="prose-hc whitespace-pre-line">{pickText(p.description, locale)}</div>
            <div className="mt-4 flex flex-wrap gap-2">
              {p.tags.map((tag) => (
                <span key={tag} className="chip chip-gold">{tag}</span>
              ))}
            </div>
          </Reveal>

          {p.verified && p.txHash && (
            <Reveal>
              <div className="card flex flex-wrap items-center gap-4 border-emerald-300/25 p-5">
                <span className="grid h-12 w-12 place-items-center rounded-xl bg-emerald-400/15 text-emerald-300">
                  <BadgeCheck size={26} />
                </span>
                <div className="min-w-0 flex-1">
                  <p className="font-semibold text-white">{t("project.onChain")}</p>
                  <p className="mt-1 font-mono text-xs text-sky-200">{shortHash(p.txHash, 12, 10)}</p>
                </div>
                <CopyButton text={p.txHash} />
              </div>
            </Reveal>
          )}

          {related.length > 0 && (
            <Reveal>
              <div className="grid gap-6 sm:grid-cols-2 xl:grid-cols-3">
                {related.map((r) => (
                  <ProjectCard key={r.slug} p={r} />
                ))}
              </div>
            </Reveal>
          )}
        </div>

        <aside className="space-y-5 lg:sticky lg:top-24 lg:self-start">
          {p.ownerName && p.ownerHandle && (
            <Link href={`/developers/${p.ownerHandle}`} className="card card-hover spotlight block p-5">
              <p className="mb-3 text-xs uppercase tracking-wider text-slate-400">{t("project.team")}</p>
              <div className="flex items-center gap-3">
                <Avatar name={p.ownerName} seed={p.ownerHandle} size={52} />
                <div>
                  <p className="font-bold text-white">{p.ownerName}</p>
                  <p className="text-sm text-gold-300">@{p.ownerHandle}</p>
                </div>
              </div>
              <p className="mt-4 text-sm font-semibold text-gold-400">{t("common.viewPortfolio")} →</p>
            </Link>
          )}
          {p.hackathonSlug && p.hackathonTitle && (
            <Link href={`/hackathons/${p.hackathonSlug}`} className="card card-hover block p-5">
              <p className="mb-2 text-xs uppercase tracking-wider text-slate-400">{t("project.fromEvent")}</p>
              <p className="flex items-center gap-2 font-bold text-white">
                <Trophy size={18} className="text-gold-400" /> {pickText(p.hackathonTitle, locale)}
              </p>
            </Link>
          )}
          <div className="card p-5 text-sm text-slate-300">
            <p className="flex justify-between"><span className="text-slate-400">{t("common.date")}</span> {formatDate(p.createdAt, locale)}</p>
          </div>
        </aside>
      </div>
    </>
  );
}
import type { Metadata } from "next";
import { ProjectForm } from "@/components/forms";
import { Container } from "@/components/PageHeader";
import { getDevelopers, getHackathons } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("form.projectTitle") };
}

export default async function NewProjectPage() {
  const [developers, hackathons] = await Promise.all([getDevelopers(), getHackathons()]);
  return (
    <Container className="py-12">
      <ProjectForm
        developers={developers.map((d) => ({ id: d.id, name: d.name, handle: d.handle }))}
        hackathons={hackathons.map((h) => ({ id: h.id, title: h.title }))}
      />
    </Container>
  );
}
import type { Metadata } from "next";
import { ProjectsExplorer } from "@/components/Explorers";
import { Container, PageHeader } from "@/components/PageHeader";
import { getProjects } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("nav.projects"), description: t("page.projectsSub") };
}

export default async function ProjectsPage() {
  const { t } = await getT();
  const projects = await getProjects();
  return (
    <>
      <PageHeader eyebrow={t("nav.projects")} title={t("page.projectsTitle")} sub={t("page.projectsSub")} />
      <Container>
        <ProjectsExplorer projects={projects} />
      </Container>
    </>
  );
}
import type { Metadata } from "next";
import { VerifyForm } from "@/components/actions";
import { Container, PageHeader } from "@/components/PageHeader";
import { getSampleHashes } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

export async function generateMetadata(): Promise<Metadata> {
  const { t } = await getT();
  return { title: t("verify.title"), description: t("verify.sub") };
}

export default async function VerifyPage({ searchParams }: { searchParams: Promise<{ hash?: string }> }) {
  const { t } = await getT();
  const { hash } = await searchParams;
  const samples = await getSampleHashes(2);
  return (
    <>
      <PageHeader eyebrow={t("nav.verify")} title={t("verify.title")} sub={t("verify.sub")} />
      <Container>
        <VerifyForm samples={samples} initialHash={typeof hash === "string" ? hash : ""} />
      </Container>
    </>
  );
}
@import "tailwindcss";

@theme {
  --color-navy-950: #09131f;
  --color-navy-900: #0d1b2e;
  --color-navy-800: #13243b;
  --color-navy-700: #1b3352;
  --color-navy-600: #284769;
  --color-gold-300: #ffe27a;
  --color-gold-400: #ffd43e;
  --color-gold-500: #f5bf1a;
  --color-gold-600: #d9a40c;
  --font-sans: ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  --font-display: ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}

:root {
  color-scheme: dark;
}

html {
  scroll-behavior: smooth;
}

body {
  background: var(--color-navy-900);
  color: #e8eef7;
  min-height: 100vh;
  overflow-x: hidden;
  -webkit-font-smoothing: antialiased;
}

::selection {
  background: var(--color-gold-400);
  color: var(--color-navy-900);
}

/* ---------- Scrollbar ---------- */
* {
  scrollbar-width: thin;
  scrollbar-color: #284769 transparent;
}

/* ---------- Background ---------- */
.bg-scene {
  position: fixed;
  inset: 0;
  z-index: -1;
  pointer-events: none;
  background:
    radial-gradient(60rem 40rem at 85% -10%, rgba(255, 212, 62, 0.1), transparent 60%),
    radial-gradient(50rem 40rem at -10% 20%, rgba(80, 140, 255, 0.12), transparent 60%),
    linear-gradient(180deg, #0d1b2e 0%, #09131f 100%);
}
.bg-grid {
  position: absolute;
  inset: 0;
  background-image:
    linear-gradient(rgba(255, 255, 255, 0.035) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.035) 1px, transparent 1px);
  background-size: 56px 56px;
  mask-image: radial-gradient(ellipse at 50% 20%, black 20%, transparent 75%);
}

/* ---------- Keyframes ---------- */
@keyframes float {
  0%, 100% { transform: translateY(0) rotate(var(--r, 0deg)); }
  50% { transform: translateY(-14px) rotate(var(--r, 0deg)); }
}
@keyframes floatSlow {
  0%, 100% { transform: translate3d(0, 0, 0) scale(1); }
  50% { transform: translate3d(30px, -20px, 0) scale(1.08); }
}
@keyframes spinSlow { to { transform: rotate(360deg); } }
@keyframes spinRev { to { transform: rotate(-360deg); } }
@keyframes pulseRing {
  0% { transform: scale(0.9); opacity: 0.7; }
  100% { transform: scale(1.8); opacity: 0; }
}
@keyframes shine {
  0% { transform: translateX(-120%) skewX(-20deg); }
  100% { transform: translateX(220%) skewX(-20deg); }
}
@keyframes gradientShift {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}
@keyframes marquee {
  from { transform: translateX(0); }
  to { transform: translateX(-50%); }
}
@keyframes blink { 50% { opacity: 0; } }
@keyframes riseIn {
  from { opacity: 0; transform: translateY(18px) scale(0.98); }
  to { opacity: 1; transform: none; }
}
@keyframes popIn {
  from { opacity: 0; transform: translateY(16px) scale(0.92); }
  to { opacity: 1; transform: none; }
}
@keyframes wave {
  0%, 100% { transform: scaleY(0.25); }
  50% { transform: scaleY(1); }
}
@keyframes orbBreath {
  0%, 100% { transform: scale(1); box-shadow: 0 0 40px rgba(255, 212, 62, 0.35); }
  50% { transform: scale(1.06); box-shadow: 0 0 70px rgba(255, 212, 62, 0.55); }
}
@keyframes drawLink {
  from { stroke-dashoffset: 900; }
  to { stroke-dashoffset: 0; }
}
@keyframes linkPulse {
  0%, 100% { filter: drop-shadow(0 0 0 rgba(255, 212, 62, 0)); }
  50% { filter: drop-shadow(0 0 22px rgba(255, 212, 62, 0.45)); }
}
@keyframes ringSlideL {
  0%, 100% { transform: translateX(0); }
  50% { transform: translateX(-12px); }
}
@keyframes ringSlideR {
  0%, 100% { transform: translateX(0); }
  50% { transform: translateX(12px); }
}

.anim-float { animation: float 6s ease-in-out infinite; }
.anim-float-slow { animation: floatSlow 14s ease-in-out infinite; }
.anim-spin-slow { animation: spinSlow 40s linear infinite; }
.anim-spin-rev { animation: spinRev 60s linear infinite; }
.anim-blink { animation: blink 1s step-end infinite; }
.anim-rise { animation: riseIn 0.7s cubic-bezier(0.2, 0.7, 0.2, 1) both; }
.anim-pop { animation: popIn 0.35s cubic-bezier(0.2, 0.8, 0.2, 1) both; }
.anim-marquee { animation: marquee 38s linear infinite; }
.marquee-wrap:hover .anim-marquee { animation-play-state: paused; }

/* ---------- Text effects ---------- */
.text-gradient {
  background: linear-gradient(100deg, #ffd43e 0%, #ffe9a0 35%, #ffffff 55%, #ffd43e 100%);
  background-size: 250% 100%;
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  animation: gradientShift 6s ease-in-out infinite;
}

/* ---------- Buttons ---------- */
.btn {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.75rem 1.4rem;
  border-radius: 9999px;
  font-weight: 600;
  font-size: 0.95rem;
  line-height: 1;
  white-space: nowrap;
  overflow: hidden;
  isolation: isolate;
  cursor: pointer;
  transition: transform 0.25s cubic-bezier(0.2, 0.8, 0.2, 1), box-shadow 0.25s, background 0.25s, border-color 0.25s, color 0.25s;
  -webkit-tap-highlight-color: transparent;
}
.btn:hover { transform: translateY(-2px); }
.btn:active { transform: translateY(0) scale(0.97); }
.btn:disabled { opacity: 0.55; cursor: not-allowed; transform: none; }
.btn svg { transition: transform 0.25s; }
.btn:hover svg.arrow { transform: translateX(4px); }

.btn-primary {
  background: linear-gradient(135deg, #ffd43e, #f5bf1a);
  color: #13243b;
  box-shadow: 0 8px 28px -6px rgba(255, 212, 62, 0.55), inset 0 1px 0 rgba(255, 255, 255, 0.5);
}
.btn-primary:hover { box-shadow: 0 14px 38px -6px rgba(255, 212, 62, 0.75), inset 0 1px 0 rgba(255, 255, 255, 0.6); }
.btn-primary::after {
  content: "";
  position: absolute;
  inset: 0;
  width: 40%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.65), transparent);
  transform: translateX(-120%) skewX(-20deg);
  z-index: -1;
}
.btn-primary:hover::after { animation: shine 0.9s ease; }

.btn-ghost {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.16);
  color: #fff;
  backdrop-filter: blur(8px);
}
.btn-ghost:hover { background: rgba(255, 255, 255, 0.12); border-color: rgba(255, 212, 62, 0.6); color: #ffe27a; }

.btn-sm { padding: 0.55rem 1rem; font-size: 0.85rem; }

/* ---------- Cards ---------- */
.card {
  position: relative;
  border-radius: 1.25rem;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.065), rgba(255, 255, 255, 0.025));
  border: 1px solid rgba(255, 255, 255, 0.09);
  backdrop-filter: blur(10px);
  transition: transform 0.35s cubic-bezier(0.2, 0.8, 0.2, 1), border-color 0.3s, box-shadow 0.35s;
  overflow: hidden;
}
.card-hover:hover {
  transform: translateY(-6px);
  border-color: rgba(255, 212, 62, 0.45);
  box-shadow: 0 24px 50px -20px rgba(0, 0, 0, 0.7), 0 0 0 1px rgba(255, 212, 62, 0.12);
}
.spotlight::before {
  content: "";
  position: absolute;
  inset: 0;
  background: radial-gradient(420px circle at var(--mx, 50%) var(--my, 0%), rgba(255, 212, 62, 0.14), transparent 45%);
  opacity: 0;
  transition: opacity 0.3s;
  pointer-events: none;
  z-index: 0;
}
.spotlight:hover::before { opacity: 1; }
.spotlight > * { position: relative; z-index: 1; }

/* ---------- Chips / badges ---------- */
.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  padding: 0.28rem 0.7rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
  color: #cfd9e8;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.1);
}
.chip-gold {
  color: #13243b;
  background: linear-gradient(135deg, #ffd43e, #ffe27a);
  border-color: transparent;
  font-weight: 700;
}
.chip-live {
  color: #7dffb2;
  background: rgba(40, 200, 120, 0.14);
  border-color: rgba(125, 255, 178, 0.35);
}
.chip-live::before {
  content: "";
  width: 6px;
  height: 6px;
  border-radius: 9999px;
  background: #4dffa0;
  box-shadow: 0 0 0 0 rgba(77, 255, 160, 0.7);
  animation: pulseDot 1.6s infinite;
}
@keyframes pulseDot {
  0% { box-shadow: 0 0 0 0 rgba(77, 255, 160, 0.7); }
  70% { box-shadow: 0 0 0 8px rgba(77, 255, 160, 0); }
  100% { box-shadow: 0 0 0 0 rgba(77, 255, 160, 0); }
}

.filter-btn {
  padding: 0.5rem 1rem;
  border-radius: 9999px;
  font-size: 0.85rem;
  font-weight: 600;
  color: #b9c7da;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  cursor: pointer;
  transition: all 0.25s;
}
.filter-btn:hover { color: #fff; border-color: rgba(255, 212, 62, 0.5); transform: translateY(-1px); }
.filter-btn[data-active="true"] {
  color: #13243b;
  background: linear-gradient(135deg, #ffd43e, #f5bf1a);
  border-color: transparent;
  box-shadow: 0 6px 20px -6px rgba(255, 212, 62, 0.6);
}

/* ---------- Inputs ---------- */
.input {
  width: 100%;
  padding: 0.75rem 1rem;
  border-radius: 0.9rem;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.12);
  color: #fff;
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
}
.input::placeholder { color: #7c8da6; }
.input:focus {
  border-color: rgba(255, 212, 62, 0.7);
  box-shadow: 0 0 0 4px rgba(255, 212, 62, 0.12);
  background: rgba(255, 255, 255, 0.08);
}
select.input option { background: #13243b; color: #fff; }
.label { display: block; margin-bottom: 0.4rem; font-size: 0.82rem; font-weight: 600; color: #b9c7da; }

/* ---------- Reveal on scroll ---------- */
.reveal {
  opacity: 0;
  transform: translateY(28px);
  transition: opacity 0.8s cubic-bezier(0.2, 0.7, 0.2, 1), transform 0.8s cubic-bezier(0.2, 0.7, 0.2, 1);
  transition-delay: var(--d, 0ms);
}
.reveal.in { opacity: 1; transform: none; }

/* ---------- Ring logo ---------- */
.ring-draw { stroke-dasharray: 900; stroke-dashoffset: 900; animation: drawLink 1.6s 0.2s ease forwards; }
.ring-glow { animation: linkPulse 4s ease-in-out infinite; }
.ring-l { animation: ringSlideL 6s ease-in-out infinite; }
.ring-r { animation: ringSlideR 6s ease-in-out infinite; }

/* ---------- Assistant ---------- */
.orb {
  position: relative;
  width: 112px;
  height: 112px;
  border-radius: 9999px;
  background: radial-gradient(circle at 35% 30%, #ffe9a0, #ffd43e 45%, #d9a40c 100%);
  color: #13243b;
  display: grid;
  place-items: center;
  cursor: pointer;
  border: none;
  transition: transform 0.2s;
}
.orb:active { transform: scale(0.94); }
.orb[data-state="idle"] { animation: orbBreath 3.2s ease-in-out infinite; }
.orb[data-state="listening"]::before,
.orb[data-state="listening"]::after,
.orb[data-state="speaking"]::before,
.orb[data-state="speaking"]::after {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: 9999px;
  border: 2px solid rgba(255, 212, 62, 0.7);
  animation: pulseRing 1.8s ease-out infinite;
}
.orb[data-state="listening"]::after,
.orb[data-state="speaking"]::after { animation-delay: 0.9s; }
.orb[data-state="thinking"] { animation: orbBreath 0.9s ease-in-out infinite; }
.wave-bar {
  width: 4px;
  height: 28px;
  border-radius: 4px;
  background: #13243b;
  transform-origin: center;
  animation: wave 0.9s ease-in-out infinite;
}

.chat-scroll::-webkit-scrollbar { width: 6px; }
.chat-scroll::-webkit-scrollbar-thumb { background: #284769; border-radius: 6px; }

.prose-hc p { margin-bottom: 1rem; line-height: 1.75; color: #c9d5e6; }

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
  .reveal { opacity: 1; transform: none; }
  .ring-draw { stroke-dashoffset: 0; }
}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" width="512" height="512">
  <defs>
    <clipPath id="lower">
      <rect x="190" y="262" width="140" height="120"/>
    </clipPath>
  </defs>
  <rect width="512" height="512" rx="112" fill="#13243b"/>
  <rect x="87" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#ffd43e" stroke-width="38"/>
  <rect x="226" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#ffffff" stroke-width="38"/>
  <g clip-path="url(#lower)">
    <rect x="87" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#ffd43e" stroke-width="38"/>
  </g>
</svg>
import type { Metadata, Viewport } from "next";
import type { ReactNode } from "react";
import "./globals.css";
import { I18nProvider } from "@/components/I18nProvider";
import { Navbar } from "@/components/Navbar";
import { Footer } from "@/components/Footer";
import { GlobalFx } from "@/components/fx";
import { AssistantWidget } from "@/components/AssistantWidget";
import { getLocale } from "@/lib/server-locale";

export const metadata: Metadata = {
  title: {
    default: "HackChain — verifiable developer reputation",
    template: "%s · HackChain",
  },
  description:
    "HackChain is a Web3 platform where hackathon wins, achievements and projects are verified by organizers and recorded on-chain — a portfolio employers can trust.",
};

export const viewport: Viewport = {
  themeColor: "#13243b",
};

export default async function RootLayout({ children }: { children: ReactNode }) {
  const locale = await getLocale();
  return (
    <html lang={locale}>
      <body className="font-sans antialiased">
        <I18nProvider initialLocale={locale}>
          <div className="bg-scene" aria-hidden="true">
            <div className="bg-grid" />
          </div>
          <GlobalFx />
          <Navbar />
          <main className="min-h-[70vh] pt-16">{children}</main>
          <Footer />
          <AssistantWidget />
        </I18nProvider>
      </body>
    </html>
  );
}
import Link from "next/link";
import { Rings } from "@/components/Logo";
import { getT } from "@/lib/server-locale";

export default async function NotFound() {
  const { t } = await getT();
  return (
    <div className="mx-auto grid max-w-xl place-items-center gap-6 px-4 py-24 text-center">
      <Rings animated className="w-56" />
      <h1 className="font-display text-6xl font-extrabold text-gradient">404</h1>
      <Link href="/" className="btn btn-primary">
        {t("nav.home")}
      </Link>
    </div>
  );
}
import Link from "next/link";
import { ArrowRight, BadgeCheck, Link2, Rocket, Trophy, Users } from "lucide-react";
import { AiPromo } from "@/components/AiPromo";
import { DeveloperCard, HackathonCard, ProjectCard } from "@/components/cards";
import { Hero } from "@/components/Hero";
import { Rings } from "@/components/Logo";
import { Reveal } from "@/components/fx";
import { getDevelopers, getHackathons, getProjects, getStats } from "@/lib/data";
import { getT } from "@/lib/server-locale";

export const dynamic = "force-dynamic";

function SectionTitle({ title, sub, href, cta }: { title: string; sub: string; href?: string; cta?: string }) {
  return (
    <Reveal>
      <div className="mb-8 flex flex-wrap items-end justify-between gap-4">
        <div>
          <h2 className="font-display text-3xl font-extrabold tracking-tight text-white sm:text-4xl">{title}</h2>
          <p className="mt-2 max-w-xl text-slate-300">{sub}</p>
        </div>
        {href && cta && (
          <Link href={href} className="group inline-flex items-center gap-2 text-sm font-semibold text-gold-400 transition hover:text-gold-300">
            {cta} <ArrowRight size={16} className="transition-transform group-hover:translate-x-1" />
          </Link>
        )}
      </div>
    </Reveal>
  );
}

export default async function HomePage() {
  const { t } = await getT();
  const [stats, hackathons, projects, developers] = await Promise.all([
    getStats(),
    getHackathons(),
    getProjects(),
    getDevelopers(),
  ]);

  const featuredHacks = hackathons.filter((h) => h.status !== "past").slice(0, 3);
  const heroHacks = featuredHacks.length ? featuredHacks : hackathons.slice(0, 3);
  const topProjects = projects.slice(0, 3);
  const topDevs = developers.slice(0, 3);

  const steps = [
    { icon: Users, title: t("home.step1t"), desc: t("home.step1d") },
    { icon: Rocket, title: t("home.step2t"), desc: t("home.step2d") },
    { icon: BadgeCheck, title: t("home.step3t"), desc: t("home.step3d") },
    { icon: Link2, title: t("home.step4t"), desc: t("home.step4d") },
  ];

  return (
    <>
      <Hero stats={stats} />

      <section className="mx-auto max-w-7xl px-4 py-16 sm:px-6 lg:px-8">
        <SectionTitle title={t("home.hackathonsTitle")} sub={t("home.hackathonsSub")} href="/hackathons" cta={t("home.viewAll")} />
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
          {heroHacks.map((h, i) => (
            <Reveal key={h.slug} delay={i * 100} className="h-full">
              <HackathonCard h={h} />
            </Reveal>
          ))}
        </div>
      </section>

      <section className="mx-auto max-w-7xl px-4 py-16 sm:px-6 lg:px-8">
        <SectionTitle title={t("home.howTitle")} sub={t("home.howSub")} />
        <div className="relative grid gap-6 md:grid-cols-2 lg:grid-cols-4">
          <div className="pointer-events-none absolute left-[12%] right-[12%] top-12 hidden h-px bg-gradient-to-r from-transparent via-gold-400/50 to-transparent lg:block" />
          {steps.map((s, i) => (
            <Reveal key={s.title} delay={i * 120} className="h-full">
              <div className="card card-hover spotlight h-full p-6">
                <div className="mb-5 flex items-center justify-between">
                  <span className="grid h-14 w-14 place-items-center rounded-2xl bg-gradient-to-br from-gold-300 to-gold-500 text-navy-800 shadow-[0_10px_30px_-8px_rgba(255,212,62,0.6)]">
                    <s.icon size={26} />
                  </span>
                  <span className="font-display text-5xl font-extrabold text-white/10">0{i + 1}</span>
                </div>
                <h3 className="font-display text-lg font-bold text-white">{s.title}</h3>
                <p className="mt-2 text-sm leading-relaxed text-slate-300">{s.desc}</p>
              </div>
            </Reveal>
          ))}
        </div>
      </section>

      <section className="mx-auto max-w-7xl px-4 py-16 sm:px-6 lg:px-8">
        <SectionTitle title={t("home.projectsTitle")} sub={t("home.projectsSub")} href="/projects" cta={t("home.viewAll")} />
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
          {topProjects.map((p, i) => (
            <Reveal key={p.slug} delay={i * 100} className="h-full">
              <ProjectCard p={p} />
            </Reveal>
          ))}
        </div>
      </section>

      <section className="mx-auto max-w-7xl px-4 py-16 sm:px-6 lg:px-8">
        <SectionTitle title={t("home.devsTitle")} sub={t("home.devsSub")} href="/developers" cta={t("home.viewAll")} />
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
          {topDevs.map((d, i) => (
            <Reveal key={d.handle} delay={i * 100} className="h-full">
              <DeveloperCard d={d} rank={i} />
            </Reveal>
          ))}
        </div>
      </section>

      <section className="mx-auto max-w-7xl px-4 py-16 sm:px-6 lg:px-8">
        <Reveal>
          <AiPromo />
        </Reveal>
      </section>

      <section className="mx-auto max-w-7xl px-4 pb-8 pt-16 sm:px-6 lg:px-8">
        <Reveal>
          <div className="relative overflow-hidden rounded-[2rem] border border-gold-400/30 bg-gradient-to-br from-navy-700 via-navy-800 to-navy-900 px-8 py-14 text-center sm:px-16">
            <Rings className="anim-float-slow pointer-events-none absolute -right-16 -top-10 w-80 opacity-[0.12]" />
            <Rings className="anim-float-slow pointer-events-none absolute -bottom-16 -left-16 w-72 opacity-[0.1]" />
            <Trophy className="mx-auto mb-5 text-gold-400" size={44} />
            <h2 className="relative font-display text-3xl font-extrabold text-white sm:text-5xl">{t("home.ctaTitle")}</h2>
            <p className="relative mx-auto mt-4 max-w-xl text-slate-300">{t("home.ctaDesc")}</p>
            <div className="relative mt-8 flex flex-wrap justify-center gap-3">
              <Link href="/developers/new" className="btn btn-primary">
                {t("nav.createProfile")} <ArrowRight size={18} className="arrow" />
              </Link>
              <Link href="/hackathons/new" className="btn btn-ghost">
                {t("nav.hostTournament")}
              </Link>
            </div>
          </div>
        </Reveal>
      </section>
    </>
  );
}
"use client";

import Link from "next/link";
import { useEffect, useState, type FormEvent } from "react";
import { BadgeCheck, Check, Copy, Heart, Loader2, ShieldCheck, ShieldX } from "lucide-react";
import { useI18n } from "./I18nProvider";
import type { AchievementKind, I18nText } from "@/lib/types";
import { formatDate } from "@/lib/ui";

export function LikeButton({ slug, initialLikes }: { slug: string; initialLikes: number }) {
  const { t } = useI18n();
  const [likes, setLikes] = useState(initialLikes);
  const [liked, setLiked] = useState(false);
  const [pop, setPop] = useState(false);

  useEffect(() => {
    setLiked(localStorage.getItem(`hc_like_${slug}`) === "1");
  }, [slug]);

  async function toggle() {
    const next = !liked;
    setLiked(next);
    setLikes((n) => n + (next ? 1 : -1));
    setPop(true);
    setTimeout(() => setPop(false), 400);
    if (next) localStorage.setItem(`hc_like_${slug}`, "1");
    else localStorage.removeItem(`hc_like_${slug}`);
    try {
      const res = await fetch(`/api/projects/${slug}/like`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ undo: !next }),
      });
      const data = (await res.json()) as { likes?: number };
      if (typeof data.likes === "number") setLikes(data.likes);
    } catch {
      /* keep optimistic value */
    }
  }

  return (
    <button
      type="button"
      onClick={toggle}
      className={`btn ${liked ? "btn-primary" : "btn-ghost"}`}
      aria-pressed={liked}
    >
      <Heart
        size={18}
        fill={liked ? "currentColor" : "none"}
        className={`transition-transform duration-300 ${pop ? "scale-150" : ""}`}
      />
      {liked ? t("project.liked") : t("project.like")} · {likes}
    </button>
  );
}

export function CopyButton({ text, label, className = "" }: { text: string; label?: string; className?: string }) {
  const { t } = useI18n();
  const [copied, setCopied] = useState(false);
  return (
    <button
      type="button"
      className={`btn btn-ghost btn-sm ${className}`}
      onClick={async () => {
        try {
          await navigator.clipboard.writeText(text.startsWith("/") ? window.location.origin + text : text);
          setCopied(true);
          setTimeout(() => setCopied(false), 1600);
        } catch {
          /* ignore */
        }
      }}
    >
      {copied ? <Check size={15} className="text-emerald-300" /> : <Copy size={15} />}
      {copied ? t("common.copied") : (label ?? t("common.copy"))}
    </button>
  );
}

interface Credential {
  title: I18nText;
  kind: AchievementKind;
  place: number | null;
  issuer: string;
  txHash: string;
  date: string;
  holderName: string;
  holderHandle: string;
  verified: boolean;
}

export function VerifyForm({ samples, initialHash = "" }: { samples: string[]; initialHash?: string }) {
  const { t, pick, locale } = useI18n();
  const [hash, setHash] = useState(initialHash);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<{ found: boolean; credential?: Credential } | null>(null);

  async function run(value: string) {
    const v = value.trim();
    if (!v) return;
    setLoading(true);
    setResult(null);
    try {
      const res = await fetch(`/api/verify?hash=${encodeURIComponent(v)}`);
      const data = (await res.json()) as { found: boolean; credential?: Credential };
      // small delay so the scanning animation is visible
      await new Promise((r) => setTimeout(r, 600));
      setResult(data);
    } catch {
      setResult({ found: false });
    }
    setLoading(false);
  }

  useEffect(() => {
    if (initialHash) void run(initialHash);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  function onSubmit(e: FormEvent) {
    e.preventDefault();
    void run(hash);
  }

  const c = result?.credential;

  return (
    <div className="mx-auto max-w-3xl space-y-6">
      <form onSubmit={onSubmit} className="card p-5 sm:p-7">
        <div className="flex flex-col gap-3 sm:flex-row">
          <input
            className="input font-mono text-sm"
            value={hash}
            onChange={(e) => setHash(e.target.value)}
            placeholder={t("verify.placeholder")}
            spellCheck={false}
          />
          <button type="submit" disabled={loading || !hash.trim()} className="btn btn-primary shrink-0">
            {loading ? <Loader2 size={18} className="animate-spin" /> : <ShieldCheck size={18} />} {t("verify.btn")}
          </button>
        </div>
        {samples.length > 0 && (
          <div className="mt-4 flex flex-wrap items-center gap-2 text-xs text-slate-400">
            <span>{t("verify.example")}:</span>
            {samples.map((s) => (
              <button
                key={s}
                type="button"
                className="chip cursor-pointer font-mono transition hover:border-gold-400/60 hover:text-gold-300"
                onClick={() => {
                  setHash(s);
                  void run(s);
                }}
              >
                {s.slice(0, 10)}…{s.slice(-6)}
              </button>
            ))}
          </div>
        )}
      </form>

      {loading && (
        <div className="card anim-pop relative grid place-items-center overflow-hidden px-6 py-14">
          <span className="absolute inset-x-0 h-24 bg-gradient-to-b from-transparent via-gold-400/20 to-transparent" style={{ animation: "float 1.2s ease-in-out infinite" }} />
          <ShieldCheck size={44} className="text-gold-400" />
        </div>
      )}

      {result && result.found && c && (
        <div className="card anim-pop border-emerald-300/30 p-6 sm:p-8">
          <div className="flex items-center gap-3 text-emerald-300">
            <BadgeCheck size={30} />
            <p className="text-lg font-bold">{t("verify.valid")}</p>
          </div>
          <h3 className="mt-5 font-display text-2xl font-extrabold text-white">{pick(c.title)}</h3>
          <dl className="mt-5 grid gap-4 text-sm sm:grid-cols-2">
            <div>
              <dt className="text-slate-400">{t("verify.holder")}</dt>
              <dd className="mt-1 font-semibold text-white">
                <Link href={`/developers/${c.holderHandle}`} className="text-gold-300 hover:underline">
                  {c.holderName}
                </Link>
              </dd>
            </div>
            <div>
              <dt className="text-slate-400">{t("verify.issuer")}</dt>
              <dd className="mt-1 font-semibold text-white">{c.issuer}</dd>
            </div>
            <div>
              <dt className="text-slate-400">{t("verify.type")}</dt>
              <dd className="mt-1 font-semibold text-white">
                {t(`kind.${c.kind}` as "kind.win")}
                {c.place ? ` · ${c.place <= 3 ? t(`place.${c.place}` as "place.1") : t("place.n", { n: c.place })}` : ""}
              </dd>
            </div>
            <div>
              <dt className="text-slate-400">{t("common.date")}</dt>
              <dd className="mt-1 font-semibold text-white">{formatDate(c.date, locale)}</dd>
            </div>
            <div className="sm:col-span-2">
              <dt className="text-slate-400">{t("verify.tx")}</dt>
              <dd className="mt-1 break-all rounded-xl bg-navy-950/70 px-3 py-2 font-mono text-xs text-sky-200">{c.txHash}</dd>
            </div>
          </dl>
        </div>
      )}

      {result && !result.found && (
        <div className="card anim-pop flex items-center gap-3 border-red-400/30 p-6 text-red-200">
          <ShieldX size={28} /> <p className="font-semibold">{t("verify.invalid")}</p>
        </div>
      )}

      <p className="text-center text-xs text-slate-500">{t("verify.note")}</p>
    </div>
  );
}
"use client";

import { Bot, MessageSquare, Mic, Sparkles } from "lucide-react";
import { useI18n } from "./I18nProvider";
import { openAssistant } from "./Navbar";

export function AiPromo() {
  const { t } = useI18n();
  return (
    <div className="card relative grid gap-10 overflow-hidden p-8 sm:p-12 lg:grid-cols-2 lg:items-center">
      <div className="anim-float-slow pointer-events-none absolute -right-20 -top-20 h-80 w-80 rounded-full bg-gold-400/15 blur-3xl" />
      <div className="relative">
        <span className="chip chip-gold mb-5">
          <Sparkles size={13} /> AI · Voice
        </span>
        <h2 className="font-display text-3xl font-extrabold text-white sm:text-4xl">{t("home.aiTitle")}</h2>
        <p className="mt-4 max-w-lg text-slate-300">{t("home.aiDesc")}</p>
        <div className="mt-7 flex flex-wrap gap-3">
          <button type="button" className="btn btn-primary" onClick={() => openAssistant("chat")}>
            <MessageSquare size={18} /> {t("home.aiChat")}
          </button>
          <button type="button" className="btn btn-ghost" onClick={() => openAssistant("voice")}>
            <Mic size={18} className="text-gold-400" /> {t("home.aiVoice")}
          </button>
        </div>
      </div>

      {/* animated mock conversation */}
      <div className="relative mx-auto w-full max-w-md space-y-3">
        <div className="anim-rise flex justify-end" style={{ animationDelay: "0.1s" }}>
          <div className="rounded-2xl rounded-br-md bg-gradient-to-br from-gold-400 to-gold-500 px-4 py-2.5 text-sm font-medium text-navy-800">
            {t("ai.s1")}
          </div>
        </div>
        <div className="anim-rise flex gap-2" style={{ animationDelay: "0.5s" }}>
          <span className="mt-1 grid h-8 w-8 shrink-0 place-items-center rounded-xl bg-white/10 text-gold-400">
            <Bot size={18} />
          </span>
          <div className="rounded-2xl rounded-bl-md border border-white/10 bg-white/[0.07] px-4 py-3 text-sm text-slate-100">
            <p>• HackChain Genesis Hackathon</p>
            <p>• Astana Code Cup 2026</p>
            <p>• Mobile Sprint 48h</p>
          </div>
        </div>
        <div className="anim-rise flex items-center justify-center gap-1.5 pt-3" style={{ animationDelay: "0.9s" }}>
          {Array.from({ length: 22 }).map((_, i) => (
            <span
              key={i}
              className="w-1 rounded-full bg-gold-400/80"
              style={{
                height: 28,
                animation: "wave 1.1s ease-in-out infinite",
                animationDelay: `${(i % 11) * 0.09}s`,
                transformOrigin: "center",
              }}
            />
          ))}
        </div>
      </div>
    </div>
  );
}
import { avatarGradient, initials } from "@/lib/ui";

export function Avatar({
  name,
  seed,
  size = 48,
  className = "",
}: {
  name: string;
  seed?: string;
  size?: number;
  className?: string;
}) {
  return (
    <span
      className={`grid shrink-0 place-items-center rounded-2xl font-extrabold text-navy-900 shadow-[inset_0_-6px_12px_rgba(0,0,0,0.15)] ${className}`}
      style={{
        width: size,
        height: size,
        fontSize: size * 0.38,
        background: avatarGradient(seed ?? name),
      }}
    >
      {initials(name)}
    </span>
  );
}
"use client";

import Link from "next/link";
import { ArrowRight, BadgeCheck, CalendarDays, Heart, MapPin, Trophy, Users } from "lucide-react";
import { useI18n } from "./I18nProvider";
import { Avatar } from "./Avatar";
import type { DictKey } from "@/lib/dict";
import type { DeveloperDTO, HackathonDTO, ProjectDTO } from "@/lib/types";
import { CATEGORY_ICON, coverFor, dateRange } from "@/lib/ui";

export function StatusChip({ status }: { status: HackathonDTO["status"] }) {
  const { t } = useI18n();
  if (status === "live") return <span className="chip chip-live">{t("common.live")}</span>;
  if (status === "upcoming") return <span className="chip chip-gold">{t("common.upcoming")}</span>;
  return <span className="chip">{t("common.past")}</span>;
}

export function HackathonCard({ h, index = 0 }: { h: HackathonDTO; index?: number }) {
  const { t, pick, locale } = useI18n();
  const pct = Math.min(100, Math.round((h.participants / Math.max(h.maxParticipants, 1)) * 100));
  const formatKey = `common.${h.format}` as DictKey;
  return (
    <Link
      href={`/hackathons/${h.slug}`}
      className="card card-hover spotlight group flex h-full flex-col"
      style={{ animationDelay: `${index * 60}ms` }}
    >
      <div className="relative h-28 overflow-hidden" style={{ background: coverFor(h.slug) }}>
        <div className="absolute inset-0 opacity-30" style={{ backgroundImage: "radial-gradient(circle at 20% 120%, #fff 0, transparent 40%), radial-gradient(circle at 90% -20%, #ffd43e 0, transparent 45%)" }} />
        <Trophy className="absolute -right-3 -top-3 h-24 w-24 rotate-12 text-white/10 transition-transform duration-500 group-hover:rotate-[20deg] group-hover:scale-110" />
        <div className="absolute left-4 top-4 flex gap-2">
          <StatusChip status={h.status} />
        </div>
        {h.prizePool && (
          <span className="absolute bottom-3 right-4 rounded-full bg-navy-900/70 px-3 py-1 text-xs font-bold text-gold-300 backdrop-blur">
            🏆 {h.prizePool}
          </span>
        )}
      </div>
      <div className="flex flex-1 flex-col p-5">
        <h3 className="font-display text-lg font-bold leading-snug text-white transition-colors group-hover:text-gold-300">
          {pick(h.title)}
        </h3>
        <p className="mt-1 text-xs text-slate-400">
          {t("common.by")} {h.organizer}
        </p>
        <div className="mt-4 space-y-1.5 text-sm text-slate-300">
          <p className="flex items-center gap-2">
            <CalendarDays size={15} className="text-gold-400" /> {dateRange(h.startsAt, h.endsAt, locale)}
          </p>
          <p className="flex items-center gap-2">
            <MapPin size={15} className="text-gold-400" /> {t(formatKey)}
            {h.location ? ` · ${h.location}` : ""}
          </p>
        </div>
        <div className="mt-4 flex flex-wrap gap-1.5">
          {h.tags.slice(0, 3).map((tag) => (
            <span key={tag} className="chip">
              {tag}
            </span>
          ))}
        </div>
        <div className="mt-auto pt-5">
          <div className="mb-1.5 flex items-center justify-between text-xs text-slate-400">
            <span className="flex items-center gap-1">
              <Users size={13} /> {t("common.registered", { n: h.participants, max: h.maxParticipants })}
            </span>
            <ArrowRight size={16} className="text-gold-400 transition-transform group-hover:translate-x-1" />
          </div>
          <div className="h-1.5 overflow-hidden rounded-full bg-white/10">
            <div className="h-full rounded-full bg-gradient-to-r from-gold-500 to-gold-300 transition-[width] duration-1000" style={{ width: `${pct}%` }} />
          </div>
        </div>
      </div>
    </Link>
  );
}

export function ProjectCard({ p }: { p: ProjectDTO }) {
  const { t, pick } = useI18n();
  return (
    <Link href={`/projects/${p.slug}`} className="card card-hover spotlight group flex h-full flex-col">
      <div className="relative h-32 overflow-hidden" style={{ background: coverFor(p.slug + p.category) }}>
        <div className="absolute inset-0 opacity-[0.12]" style={{ backgroundImage: "linear-gradient(#fff 1px, transparent 1px), linear-gradient(90deg, #fff 1px, transparent 1px)", backgroundSize: "22px 22px" }} />
        <span className="absolute right-5 top-4 text-5xl drop-shadow-lg transition-transform duration-500 group-hover:-translate-y-1 group-hover:rotate-6 group-hover:scale-110">
          {CATEGORY_ICON[p.category] ?? "💡"}
        </span>
        <div className="absolute left-4 top-4 flex flex-wrap gap-2">
          <span className={`chip ${p.scale === "large" ? "chip-gold" : ""}`}>
            {p.scale === "large" ? t("common.large") : t("common.small")}
          </span>
          {p.verified && (
            <span className="chip border-emerald-300/40 bg-emerald-400/15 text-emerald-200">
              <BadgeCheck size={12} /> {t("common.verified")}
            </span>
          )}
        </div>
      </div>
      <div className="flex flex-1 flex-col p-5">
        <h3 className="font-display text-lg font-bold text-white transition-colors group-hover:text-gold-300">{pick(p.title)}</h3>
        <p className="mt-2 line-clamp-2 text-sm leading-relaxed text-slate-300">{pick(p.summary)}</p>
        <div className="mt-3 flex flex-wrap gap-1.5">
          {p.tags.slice(0, 3).map((tag) => (
            <span key={tag} className="chip">
              {tag}
            </span>
          ))}
        </div>
        <div className="mt-auto flex items-center justify-between pt-5 text-sm">
          <span className="flex min-w-0 items-center gap-2 text-slate-300">
            {p.ownerName && <Avatar name={p.ownerName} seed={p.ownerHandle ?? p.ownerName} size={26} className="!rounded-lg" />}
            <span className="truncate">{p.ownerName}</span>
          </span>
          <span className="flex items-center gap-1 text-slate-400 transition-colors group-hover:text-rose-300">
            <Heart size={15} /> {p.likes}
          </span>
        </div>
      </div>
    </Link>
  );
}

export function DeveloperCard({ d, rank }: { d: DeveloperDTO; rank?: number }) {
  const { t, pick } = useI18n();
  return (
    <Link href={`/developers/${d.handle}`} className="card card-hover spotlight group flex h-full flex-col p-5">
      <div className="flex items-start gap-4">
        <div className="relative">
          <Avatar name={d.name} seed={d.handle} size={60} className="transition-transform duration-300 group-hover:scale-105 group-hover:-rotate-3" />
          {d.openToWork && (
            <span title={t("common.openToWork")} className="absolute -bottom-1 -right-1 h-4 w-4 rounded-full border-2 border-navy-800 bg-emerald-400" />
          )}
        </div>
        <div className="min-w-0 flex-1">
          <div className="flex items-center gap-2">
            <h3 className="truncate font-display text-lg font-bold text-white transition-colors group-hover:text-gold-300">{d.name}</h3>
            {rank !== undefined && rank < 3 && <span className="text-lg">{["🥇", "🥈", "🥉"][rank]}</span>}
          </div>
          <p className="truncate text-sm text-gold-300">{pick(d.role)}</p>
          {d.location && (
            <p className="mt-0.5 flex items-center gap-1 truncate text-xs text-slate-400">
              <MapPin size={12} /> {d.location}
            </p>
          )}
        </div>
      </div>
      <p className="mt-4 line-clamp-2 text-sm leading-relaxed text-slate-300">{pick(d.bio)}</p>
      <div className="mt-4 flex flex-wrap gap-1.5">
        {d.skills.slice(0, 4).map((s) => (
          <span key={s} className="chip">
            {s}
          </span>
        ))}
      </div>
      <div className="mt-auto grid grid-cols-3 gap-2 pt-5 text-center">
        <Stat value={d.reputation} label={t("common.reputation")} gold />
        <Stat value={d.winsCount} label={t("dev.wins")} />
        <Stat value={d.projectsCount} label={t("nav.projects")} />
      </div>
    </Link>
  );
}

function Stat({ value, label, gold = false }: { value: number; label: string; gold?: boolean }) {
  return (
    <div className="rounded-xl border border-white/8 bg-white/5 px-2 py-2">
      <p className={`text-lg font-extrabold leading-none ${gold ? "text-gold-400" : "text-white"}`}>{value}</p>
      <p className="mt-1 truncate text-[10px] uppercase tracking-wide text-slate-400">{label}</p>
    </div>
  );
}
"use client";

import Link from "next/link";
import { useMemo, useState } from "react";
import { Plus, Search, SearchX } from "lucide-react";
import { useI18n } from "./I18nProvider";
import { DeveloperCard, HackathonCard, ProjectCard } from "./cards";
import type { DictKey } from "@/lib/dict";
import type { DeveloperDTO, HackathonDTO, HackathonStatus, ProjectDTO, ProjectScale } from "@/lib/types";
import { CATEGORY_ICON } from "@/lib/ui";

function SearchBox({
  value,
  onChange,
  placeholder,
}: {
  value: string;
  onChange: (v: string) => void;
  placeholder: string;
}) {
  return (
    <label className="relative block w-full md:max-w-md">
      <Search size={18} className="pointer-events-none absolute left-4 top-1/2 -translate-y-1/2 text-slate-400" />
      <input
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder}
        className="input !rounded-full !py-3 !pl-11"
        aria-label={placeholder}
      />
    </label>
  );
}

function Empty() {
  const { t } = useI18n();
  return (
    <div className="card anim-rise grid place-items-center gap-3 px-6 py-16 text-center">
      <SearchX size={40} className="text-gold-400" />
      <p className="text-slate-300">{t("common.noResults")}</p>
    </div>
  );
}

function ResultsCount({ n }: { n: number }) {
  const { t } = useI18n();
  return <p className="text-sm text-slate-400">{t("common.results", { n })}</p>;
}

function matches(haystack: string[], q: string) {
  if (!q) return true;
  const needle = q.toLowerCase();
  return haystack.some((h) => h.toLowerCase().includes(needle));
}

/* ------------------------------ Hackathons ------------------------------ */

export function HackathonsExplorer({ hackathons }: { hackathons: HackathonDTO[] }) {
  const { t } = useI18n();
  const [q, setQ] = useState("");
  const [status, setStatus] = useState<"all" | HackathonStatus>("all");

  const filtered = useMemo(
    () =>
      hackathons.filter(
        (h) =>
          (status === "all" || h.status === status) &&
          matches([h.title.en, h.title.ru, h.title.kk, h.organizer, h.location, ...h.tags], q.trim()),
      ),
    [hackathons, q, status],
  );

  const filters: { id: "all" | HackathonStatus; label: DictKey }[] = [
    { id: "all", label: "common.all" },
    { id: "live", label: "common.live" },
    { id: "upcoming", label: "common.upcoming" },
    { id: "past", label: "common.past" },
  ];

  return (
    <div className="space-y-6">
      <div className="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <SearchBox value={q} onChange={setQ} placeholder={t("search.hackathons")} />
        <div className="flex flex-wrap gap-2">
          {filters.map((f) => (
            <button key={f.id} type="button" className="filter-btn" data-active={status === f.id} onClick={() => setStatus(f.id)}>
              {t(f.label)}
            </button>
          ))}
        </div>
      </div>
      <div className="flex items-center justify-between">
        <ResultsCount n={filtered.length} />
        <Link href="/hackathons/new" className="btn btn-primary btn-sm">
          <Plus size={16} /> {t("nav.hostTournament")}
        </Link>
      </div>
      {filtered.length ? (
        <div className="grid gap-6 sm:grid-cols-2 lg:grid-cols-3">
          {filtered.map((h, i) => (
            <div key={h.slug} className="anim-rise" style={{ animationDelay: `${Math.min(i, 8) * 60}ms` }}>
              <HackathonCard h={h} />
            </div>
          ))}
        </div>
      ) : (
        <Empty />
      )}
    </div>
  );
}

/* ------------------------------- Projects ------------------------------- */

export function ProjectsExplorer({ projects }: { projects: ProjectDTO[] }) {
  const { t } = useI18n();
  const [q, setQ] = useState("");
  const [scale, setScale] = useState<"all" | ProjectScale>("all");
  const [category, setCategory] = useState<string>("all");
  const [sort, setSort] = useState<"likes" | "new">("likes");

  const categories = useMemo(() => Array.from(new Set(projects.map((p) => p.category))), [projects]);

  const filtered = useMemo(() => {
    const list = projects.filter(
      (p) =>
        (scale === "all" || p.scale === scale) &&
        (category === "all" || p.category === category) &&
        matches([p.title.en, p.summary.en, p.summary.ru, p.summary.kk, p.ownerName ?? "", ...p.tags], q.trim()),
    );
    return [...list].sort((a, b) =>
      sort === "likes" ? b.likes - a.likes : new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
    );
  }, [projects, q, scale, category, sort]);

  return (
    <div className="space-y-6">
      <div className="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <SearchBox value={q} onChange={setQ} placeholder={t("search.projects")} />
        <div className="flex flex-wrap gap-2">
          {(["all", "large", "small"] as const).map((s) => (
            <button key={s} type="button" className="filter-btn" data-active={scale === s} onClick={() => setScale(s)}>
              {s === "all" ? t("common.all") : s === "large" ? t("common.large") : t("common.small")}
            </button>
          ))}
        </div>
      </div>
      <div className="-mx-1 flex gap-2 overflow-x-auto px-1 pb-1">
        <button type="button" className="filter-btn shrink-0" data-active={category === "all"} onClick={() => setCategory("all")}>
          {t("common.all")}
        </button>
        {categories.map((c) => (
          <button key={c} type="button" className="filter-btn shrink-0" data-active={category === c} onClick={() => setCategory(c)}>
            {CATEGORY_ICON[c]} {t(`cat.${c}` as DictKey)}
          </button>
        ))}
      </div>
      <div className="flex flex-wrap items-center justify-between gap-3">
        <ResultsCount n={filtered.length} />
        <div className="flex items-center gap-3">
          <select value={sort} onChange={(e) => setSort(e.target.value as "likes" | "new")} className="input !w-auto !rounded-full !py-2 text-sm" aria-label="Sort">
            <option value="likes">{t("common.sortLikes")}</option>
            <option value="new">{t("common.sortNew")}</option>
          </select>
          <Link href="/projects/new" className="btn btn-primary btn-sm">
            <Plus size={16} /> {t("form.publish")}
          </Link>
        </div>
      </div>
      {filtered.length ? (
        <div className="grid gap-6 sm:grid-cols-2 lg:grid-cols-3">
          {filtered.map((p, i) => (
            <div key={p.slug} className="anim-rise" style={{ animationDelay: `${Math.min(i, 8) * 55}ms` }}>
              <ProjectCard p={p} />
            </div>
          ))}
        </div>
      ) : (
        <Empty />
      )}
    </div>
  );
}

/* ------------------------------ Developers ------------------------------ */

export function DevelopersExplorer({ developers }: { developers: DeveloperDTO[] }) {
  const { t } = useI18n();
  const [q, setQ] = useState("");
  const [skill, setSkill] = useState<string>("all");
  const [openOnly, setOpenOnly] = useState(false);

  const topSkills = useMemo(() => {
    const counts = new Map<string, number>();
    for (const d of developers) for (const s of d.skills) counts.set(s, (counts.get(s) ?? 0) + 1);
    return [...counts.entries()].sort((a, b) => b[1] - a[1]).slice(0, 10).map(([s]) => s);
  }, [developers]);

  const filtered = useMemo(
    () =>
      developers.filter(
        (d) =>
          (skill === "all" || d.skills.includes(skill)) &&
          (!openOnly || d.openToWork) &&
          matches([d.name, d.handle, d.location, d.role.en, d.role.ru, d.role.kk, ...d.skills], q.trim()),
      ),
    [developers, q, skill, openOnly],
  );

  return (
    <div className="space-y-6">
      <div className="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <SearchBox value={q} onChange={setQ} placeholder={t("search.devs")} />
        <button type="button" className="filter-btn" data-active={openOnly} onClick={() => setOpenOnly((v) => !v)}>
          ● {t("common.openToWork")}
        </button>
      </div>
      <div className="-mx-1 flex gap-2 overflow-x-auto px-1 pb-1">
        <button type="button" className="filter-btn shrink-0" data-active={skill === "all"} onClick={() => setSkill("all")}>
          {t("common.all")}
        </button>
        {topSkills.map((s) => (
          <button key={s} type="button" className="filter-btn shrink-0" data-active={skill === s} onClick={() => setSkill(s)}>
            {s}
          </button>
        ))}
      </div>
      <div className="flex items-center justify-between">
        <ResultsCount n={filtered.length} />
        <Link href="/developers/new" className="btn btn-primary btn-sm">
          <Plus size={16} /> {t("nav.createProfile")}
        </Link>
      </div>
      {filtered.length ? (
        <div className="grid gap-6 sm:grid-cols-2 lg:grid-cols-3">
          {filtered.map((d, i) => (
            <div key={d.handle} className="anim-rise" style={{ animationDelay: `${Math.min(i, 8) * 55}ms` }}>
              <DeveloperCard d={d} rank={q || skill !== "all" || openOnly ? undefined : i} />
            </div>
          ))}
        </div>
      ) : (
        <Empty />
      )}
    </div>
  );
}
"use client";

import Link from "next/link";
import { useI18n } from "./I18nProvider";
import { LangSwitch } from "./Navbar";
import { Wordmark } from "./Logo";

export function Footer() {
  const { t } = useI18n();
  return (
    <footer className="relative mt-24 border-t border-white/10 bg-navy-950/60">
      <div className="mx-auto grid max-w-7xl gap-10 px-4 py-14 sm:px-6 md:grid-cols-[1.4fr_1fr_1fr_1fr] lg:px-8">
        <div className="space-y-4">
          <Wordmark />
          <p className="max-w-xs text-sm leading-relaxed text-slate-400">{t("footer.tagline")}</p>
        </div>
        <div>
          <h4 className="mb-4 text-sm font-semibold uppercase tracking-wider text-gold-400">{t("footer.platform")}</h4>
          <ul className="space-y-2.5 text-sm text-slate-300">
            <li><Link className="transition hover:text-white" href="/hackathons">{t("nav.hackathons")}</Link></li>
            <li><Link className="transition hover:text-white" href="/projects">{t("nav.projects")}</Link></li>
            <li><Link className="transition hover:text-white" href="/developers">{t("nav.developers")}</Link></li>
            <li><Link className="transition hover:text-white" href="/verify">{t("nav.verify")}</Link></li>
          </ul>
        </div>
        <div>
          <h4 className="mb-4 text-sm font-semibold uppercase tracking-wider text-gold-400">{t("footer.community")}</h4>
          <ul className="space-y-2.5 text-sm text-slate-300">
            <li><Link className="transition hover:text-white" href="/hackathons/new">{t("nav.hostTournament")}</Link></li>
            <li><Link className="transition hover:text-white" href="/developers/new">{t("nav.createProfile")}</Link></li>
            <li><Link className="transition hover:text-white" href="/projects/new">{t("form.projectTitle")}</Link></li>
          </ul>
        </div>
        <div>
          <h4 className="mb-4 text-sm font-semibold uppercase tracking-wider text-gold-400">{t("footer.languages")}</h4>
          <LangSwitch />
        </div>
      </div>
      <div className="border-t border-white/10 py-5 text-center text-xs text-slate-500">
        © {new Date().getFullYear()} HackChain. {t("footer.rights")}
      </div>
    </footer>
  );
}
"use client";

import Link from "next/link";
import { useRouter } from "next/navigation";
import { useEffect, useState, type FormEvent, type ReactNode } from "react";
import { AlertCircle, CheckCircle2, Loader2, Rocket, Sparkles, Trophy, UserPlus } from "lucide-react";
import { useI18n } from "./I18nProvider";
import type { DictKey } from "@/lib/dict";
import type { HackathonStatus, I18nText } from "@/lib/types";
import { CATEGORIES, CATEGORY_ICON } from "@/lib/ui";

async function postJson(url: string, body: unknown): Promise<{ ok: boolean; data: Record<string, unknown> }> {
  try {
    const res = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    const data = (await res.json().catch(() => ({}))) as Record<string, unknown>;
    return { ok: res.ok, data };
  } catch {
    return { ok: false, data: { error: "generic" } };
  }
}

function useErrorText() {
  const { t } = useI18n();
  return (code: unknown): string => {
    switch (code) {
      case "required":
        return t("form.required");
      case "email":
        return t("err.email");
      case "handleTaken":
        return t("err.handleTaken");
      case "dates":
        return t("err.dates");
      case "full":
        return t("reg.full");
      case "closed":
        return t("reg.closed");
      case "already":
        return t("reg.already");
      default:
        return t("err.generic");
    }
  };
}

function Field({
  label,
  hint,
  children,
  className = "",
}: {
  label: string;
  hint?: string;
  children: ReactNode;
  className?: string;
}) {
  return (
    <div className={className}>
      <label className="label">{label}</label>
      {children}
      {hint && <p className="mt-1.5 text-xs text-slate-500">{hint}</p>}
    </div>
  );
}

function FormCard({
  icon,
  title,
  sub,
  children,
}: {
  icon: ReactNode;
  title: string;
  sub: string;
  children: ReactNode;
}) {
  return (
    <div className="card anim-rise mx-auto max-w-3xl p-6 sm:p-9">
      <div className="mb-8 flex items-start gap-4">
        <span className="grid h-14 w-14 shrink-0 place-items-center rounded-2xl bg-gradient-to-br from-gold-300 to-gold-500 text-navy-800 shadow-[0_10px_30px_-8px_rgba(255,212,62,0.6)]">
          {icon}
        </span>
        <div>
          <h2 className="font-display text-2xl font-extrabold text-white sm:text-3xl">{title}</h2>
          <p className="mt-1 text-slate-300">{sub}</p>
        </div>
      </div>
      {children}
    </div>
  );
}

function Submit({ loading, label, done }: { loading: boolean; label: string; done: boolean }) {
  const { t } = useI18n();
  return (
    <button type="submit" disabled={loading || done} className="btn btn-primary w-full !py-3.5 text-base sm:w-auto sm:min-w-56">
      {loading ? (
        <>
          <Loader2 size={18} className="animate-spin" /> {t("form.saving")}
        </>
      ) : done ? (
        <>
          <CheckCircle2 size={18} /> {t("form.success")}
        </>
      ) : (
        <>
          <Sparkles size={18} /> {label}
        </>
      )}
    </button>
  );
}

function ErrorBox({ message }: { message: string }) {
  if (!message) return null;
  return (
    <p className="anim-pop flex items-center gap-2 rounded-xl border border-red-400/30 bg-red-500/10 px-4 py-3 text-sm text-red-200">
      <AlertCircle size={16} /> {message}
    </p>
  );
}

/* ----------------------------- Profile form ----------------------------- */

export function ProfileForm() {
  const { t } = useI18n();
  const router = useRouter();
  const errText = useErrorText();
  const [f, setF] = useState({
    name: "",
    handle: "",
    role: "",
    bio: "",
    skills: "",
    location: "",
    github: "",
    website: "",
    telegram: "",
    wallet: "",
    openToWork: true,
  });
  const [loading, setLoading] = useState(false);
  const [done, setDone] = useState(false);
  const [error, setError] = useState("");
  const set = (k: keyof typeof f) => (e: { target: { value: string } }) => setF((s) => ({ ...s, [k]: e.target.value }));
  const skillChips = f.skills.split(",").map((s) => s.trim()).filter(Boolean).slice(0, 14);

  async function submit(e: FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);
    const { ok, data } = await postJson("/api/developers", f);
    setLoading(false);
    if (!ok) return setError(errText(data.error));
    setDone(true);
    router.push(`/developers/${String(data.handle)}`);
    router.refresh();
  }

  return (
    <FormCard icon={<UserPlus size={28} />} title={t("form.profileTitle")} sub={t("form.profileSub")}>
      <form onSubmit={submit} className="space-y-5">
        <div className="grid gap-5 sm:grid-cols-2">
          <Field label={`${t("form.name")} *`}>
            <input className="input" value={f.name} onChange={set("name")} required maxLength={80} />
          </Field>
          <Field label={t("form.handle")} hint={t("form.handleHint")}>
            <input className="input" value={f.handle} onChange={set("handle")} maxLength={30} placeholder="aruzhan_dev" />
          </Field>
        </div>
        <Field label={`${t("form.role")} *`}>
          <input className="input" value={f.role} onChange={set("role")} required maxLength={80} placeholder={t("form.rolePlaceholder")} />
        </Field>
        <Field label={`${t("form.bio")} *`}>
          <textarea className="input min-h-28 resize-y" value={f.bio} onChange={set("bio")} required maxLength={800} placeholder={t("form.bioPlaceholder")} />
        </Field>
        <Field label={t("form.skills")} hint={t("form.skillsHint")}>
          <input className="input" value={f.skills} onChange={set("skills")} placeholder="React, Solidity, Python" />
          {skillChips.length > 0 && (
            <div className="mt-2 flex flex-wrap gap-1.5">
              {skillChips.map((s) => (
                <span key={s} className="chip chip-gold anim-pop">
                  {s}
                </span>
              ))}
            </div>
          )}
        </Field>
        <div className="grid gap-5 sm:grid-cols-2">
          <Field label={t("form.location")}>
            <input className="input" value={f.location} onChange={set("location")} maxLength={80} />
          </Field>
          <Field label={t("form.telegram")}>
            <input className="input" value={f.telegram} onChange={set("telegram")} maxLength={40} />
          </Field>
          <Field label={t("form.github")}>
            <input className="input" value={f.github} onChange={set("github")} placeholder="github.com/username" />
          </Field>
          <Field label={t("form.website")}>
            <input className="input" value={f.website} onChange={set("website")} placeholder="example.com" />
          </Field>
        </div>
        <Field label={t("form.wallet")}>
          <input className="input font-mono text-sm" value={f.wallet} onChange={set("wallet")} placeholder="0x…" maxLength={66} />
        </Field>
        <label className="flex cursor-pointer items-center gap-3 text-sm text-slate-200">
          <input
            type="checkbox"
            checked={f.openToWork}
            onChange={(e) => setF((s) => ({ ...s, openToWork: e.target.checked }))}
            className="h-5 w-5 cursor-pointer accent-yellow-400"
          />
          {t("form.openToWork")}
        </label>
        <ErrorBox message={error} />
        <Submit loading={loading} done={done} label={t("form.createProfile")} />
      </form>
    </FormCard>
  );
}

/* ----------------------------- Project form ----------------------------- */

export function ProjectForm({
  developers,
  hackathons,
}: {
  developers: { id: number; name: string; handle: string }[];
  hackathons: { id: number; title: I18nText }[];
}) {
  const { t, pick } = useI18n();
  const router = useRouter();
  const errText = useErrorText();
  const [f, setF] = useState({
    title: "",
    summary: "",
    description: "",
    scale: "small" as "small" | "large",
    category: "web",
    tags: "",
    repoUrl: "",
    demoUrl: "",
    ownerId: "",
    hackathonId: "",
  });
  const [loading, setLoading] = useState(false);
  const [done, setDone] = useState(false);
  const [error, setError] = useState("");
  const set = (k: keyof typeof f) => (e: { target: { value: string } }) => setF((s) => ({ ...s, [k]: e.target.value }));

  async function submit(e: FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);
    const { ok, data } = await postJson("/api/projects", {
      ...f,
      ownerId: Number(f.ownerId),
      hackathonId: f.hackathonId ? Number(f.hackathonId) : null,
    });
    setLoading(false);
    if (!ok) return setError(errText(data.error));
    setDone(true);
    router.push(`/projects/${String(data.slug)}`);
    router.refresh();
  }

  return (
    <FormCard icon={<Rocket size={28} />} title={t("form.projectTitle")} sub={t("form.projectSub")}>
      <form onSubmit={submit} className="space-y-5">
        <div className="grid gap-5 sm:grid-cols-2">
          <Field label={`${t("form.projectName")} *`}>
            <input className="input" value={f.title} onChange={set("title")} required maxLength={80} />
          </Field>
          <Field label={`${t("form.author")} *`}>
            <select className="input" value={f.ownerId} onChange={set("ownerId")} required>
              <option value="">{t("form.selectAuthor")}</option>
              {developers.map((d) => (
                <option key={d.id} value={d.id}>
                  {d.name} (@{d.handle})
                </option>
              ))}
            </select>
            <p className="mt-1.5 text-xs text-slate-500">
              {t("form.noProfile")}{" "}
              <Link href="/developers/new" className="font-semibold text-gold-400 hover:underline">
                {t("form.createFirst")}
              </Link>
            </p>
          </Field>
        </div>
        <Field label={`${t("form.summary")} *`}>
          <input className="input" value={f.summary} onChange={set("summary")} required maxLength={200} />
        </Field>
        <Field label={`${t("form.description")} *`}>
          <textarea className="input min-h-32 resize-y" value={f.description} onChange={set("description")} required maxLength={2500} />
        </Field>

        <div>
          <label className="label">{t("form.scale")}</label>
          <div className="grid gap-3 sm:grid-cols-2">
            {(["small", "large"] as const).map((s) => (
              <button
                key={s}
                type="button"
                onClick={() => setF((x) => ({ ...x, scale: s }))}
                className={`cursor-pointer rounded-2xl border p-4 text-left transition-all duration-300 hover:-translate-y-0.5 ${
                  f.scale === s
                    ? "border-gold-400 bg-gold-400/10 shadow-[0_8px_30px_-10px_rgba(255,212,62,0.5)]"
                    : "border-white/12 bg-white/5 hover:border-white/30"
                }`}
              >
                <p className="font-bold text-white">{s === "small" ? `⚡ ${t("common.small")}` : `🏗️ ${t("common.large")}`}</p>
              </button>
            ))}
          </div>
        </div>

        <div className="grid gap-5 sm:grid-cols-2">
          <Field label={t("form.category")}>
            <select className="input" value={f.category} onChange={set("category")}>
              {CATEGORIES.map((c) => (
                <option key={c} value={c}>
                  {CATEGORY_ICON[c]} {t(`cat.${c}` as DictKey)}
                </option>
              ))}
            </select>
          </Field>
          <Field label={t("form.hackathonOpt")}>
            <select className="input" value={f.hackathonId} onChange={set("hackathonId")}>
              <option value="">{t("form.none")}</option>
              {hackathons.map((h) => (
                <option key={h.id} value={h.id}>
                  {pick(h.title)}
                </option>
              ))}
            </select>
          </Field>
          <Field label={t("form.repoUrl")}>
            <input className="input" value={f.repoUrl} onChange={set("repoUrl")} placeholder="github.com/you/project" />
          </Field>
          <Field label={t("form.demoUrl")}>
            <input className="input" value={f.demoUrl} onChange={set("demoUrl")} placeholder="project.example.com" />
          </Field>
        </div>
        <Field label={t("form.tags")} hint={t("form.tagsHint")}>
          <input className="input" value={f.tags} onChange={set("tags")} placeholder="React, Solidity, AI" />
        </Field>
        <ErrorBox message={error} />
        <Submit loading={loading} done={done} label={t("form.publish")} />
      </form>
    </FormCard>
  );
}

/* ---------------------------- Hackathon form ---------------------------- */

function toLocalInput(d: Date) {
  const p = (n: number) => String(n).padStart(2, "0");
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}T${p(d.getHours())}:00`;
}

export function HackathonForm() {
  const { t } = useI18n();
  const router = useRouter();
  const errText = useErrorText();
  const [f, setF] = useState({
    title: "",
    description: "",
    organizer: "",
    format: "online",
    location: "",
    startsAt: "",
    endsAt: "",
    prizePool: "",
    maxParticipants: "100",
    tags: "",
  });
  const [loading, setLoading] = useState(false);
  const [done, setDone] = useState(false);
  const [error, setError] = useState("");
  const set = (k: keyof typeof f) => (e: { target: { value: string } }) => setF((s) => ({ ...s, [k]: e.target.value }));

  useEffect(() => {
    const start = new Date(Date.now() + 14 * 86400000);
    start.setHours(10, 0, 0, 0);
    const end = new Date(start.getTime() + 2 * 86400000);
    setF((s) => ({ ...s, startsAt: toLocalInput(start), endsAt: toLocalInput(end) }));
  }, []);

  async function submit(e: FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);
    const { ok, data } = await postJson("/api/hackathons", {
      ...f,
      startsAt: new Date(f.startsAt).toISOString(),
      endsAt: new Date(f.endsAt).toISOString(),
      maxParticipants: Number(f.maxParticipants),
    });
    setLoading(false);
    if (!ok) return setError(errText(data.error));
    setDone(true);
    router.push(`/hackathons/${String(data.slug)}`);
    router.refresh();
  }

  return (
    <FormCard icon={<Trophy size={28} />} title={t("form.hackTitle")} sub={t("form.hackSub")}>
      <form onSubmit={submit} className="space-y-5">
        <Field label={`${t("form.eventName")} *`}>
          <input className="input" value={f.title} onChange={set("title")} required maxLength={90} />
        </Field>
        <Field label={`${t("form.eventDesc")} *`}>
          <textarea className="input min-h-36 resize-y" value={f.description} onChange={set("description")} required maxLength={3000} />
        </Field>
        <div className="grid gap-5 sm:grid-cols-2">
          <Field label={`${t("form.organizer")} *`}>
            <input className="input" value={f.organizer} onChange={set("organizer")} required maxLength={80} />
          </Field>
          <Field label={t("form.format")}>
            <select className="input" value={f.format} onChange={set("format")}>
              <option value="online">{t("common.online")}</option>
              <option value="offline">{t("common.offline")}</option>
              <option value="hybrid">{t("common.hybrid")}</option>
            </select>
          </Field>
          <Field label={t("form.start")}>
            <input type="datetime-local" className="input" value={f.startsAt} onChange={set("startsAt")} required />
          </Field>
          <Field label={t("form.end")}>
            <input type="datetime-local" className="input" value={f.endsAt} onChange={set("endsAt")} required />
          </Field>
          <Field label={t("form.city")}>
            <input className="input" value={f.location} onChange={set("location")} maxLength={100} />
          </Field>
          <Field label={t("form.max")}>
            <input type="number" min={2} max={100000} className="input" value={f.maxParticipants} onChange={set("maxParticipants")} />
          </Field>
          <Field label={t("form.prize")}>
            <input className="input" value={f.prizePool} onChange={set("prizePool")} placeholder={t("form.prizePlaceholder")} maxLength={60} />
          </Field>
          <Field label={t("form.tags")} hint={t("form.tagsHint")}>
            <input className="input" value={f.tags} onChange={set("tags")} placeholder="AI, Web3, Mobile" />
          </Field>
        </div>
        <ErrorBox message={error} />
        <Submit loading={loading} done={done} label={t("form.createEvent")} />
      </form>
    </FormCard>
  );
}

/* --------------------------- Registration form --------------------------- */

export function RegisterForm({
  slug,
  status,
  participants,
  max,
}: {
  slug: string;
  status: HackathonStatus;
  participants: number;
  max: number;
}) {
  const { t } = useI18n();
  const router = useRouter();
  const errText = useErrorText();
  const [f, setF] = useState({ name: "", email: "", teamName: "", handle: "", experience: "middle" });
  const [loading, setLoading] = useState(false);
  const [done, setDone] = useState(false);
  const [error, setError] = useState("");
  const set = (k: keyof typeof f) => (e: { target: { value: string } }) => setF((s) => ({ ...s, [k]: e.target.value }));

  if (status === "past") {
    return <p className="rounded-xl border border-white/10 bg-white/5 px-4 py-3 text-sm text-slate-300">{t("reg.closed")}</p>;
  }
  if (participants >= max && !done) {
    return <p className="rounded-xl border border-white/10 bg-white/5 px-4 py-3 text-sm text-slate-300">{t("reg.full")}</p>;
  }

  async function submit(e: FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);
    const { ok, data } = await postJson(`/api/hackathons/${slug}/register`, f);
    setLoading(false);
    if (!ok) return setError(errText(data.error));
    setDone(true);
    router.refresh();
  }

  if (done) {
    return (
      <div className="anim-pop grid place-items-center gap-3 rounded-2xl border border-emerald-300/30 bg-emerald-400/10 px-5 py-10 text-center">
        <CheckCircle2 size={44} className="text-emerald-300" />
        <p className="font-semibold text-emerald-100">{t("reg.success")}</p>
      </div>
    );
  }

  const levels: { id: string; label: DictKey }[] = [
    { id: "beginner", label: "reg.beginner" },
    { id: "middle", label: "reg.middle" },
    { id: "pro", label: "reg.pro" },
  ];

  return (
    <form onSubmit={submit} className="space-y-4">
      <Field label={`${t("reg.name")} *`}>
        <input className="input" value={f.name} onChange={set("name")} required maxLength={80} />
      </Field>
      <Field label={`${t("reg.email")} *`}>
        <input type="email" className="input" value={f.email} onChange={set("email")} required maxLength={120} />
      </Field>
      <Field label={t("reg.team")}>
        <input className="input" value={f.teamName} onChange={set("teamName")} maxLength={60} />
      </Field>
      <Field label={t("reg.handle")}>
        <input className="input" value={f.handle} onChange={set("handle")} maxLength={30} placeholder="aruzhan" />
      </Field>
      <div>
        <label className="label">{t("reg.level")}</label>
        <div className="grid grid-cols-3 gap-2">
          {levels.map((l) => (
            <button
              key={l.id}
              type="button"
              className="filter-btn !px-2 text-center"
              data-active={f.experience === l.id}
              onClick={() => setF((s) => ({ ...s, experience: l.id }))}
            >
              {t(l.label)}
            </button>
          ))}
        </div>
      </div>
      <ErrorBox message={error} />
      <button type="submit" disabled={loading} className="btn btn-primary w-full !py-3.5">
        {loading ? (
          <>
            <Loader2 size={18} className="animate-spin" /> {t("reg.sending")}
          </>
        ) : (
          <>
            <Trophy size={18} /> {t("reg.btn")}
          </>
        )}
      </button>
    </form>
  );
}
"use client";

import { useEffect, useRef, useState, type CSSProperties, type ReactNode } from "react";

/** Fade/slide children in when scrolled into view. */
export function Reveal({
  children,
  delay = 0,
  className = "",
}: {
  children: ReactNode;
  delay?: number;
  className?: string;
}) {
  const ref = useRef<HTMLDivElement>(null);
  const [shown, setShown] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const io = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setShown(true);
          io.disconnect();
        }
      },
      { threshold: 0.1, rootMargin: "0px 0px -30px 0px" },
    );
    io.observe(el);
    return () => io.disconnect();
  }, []);

  return (
    <div
      ref={ref}
      className={`reveal ${shown ? "in" : ""} ${className}`}
      style={{ "--d": `${delay}ms` } as CSSProperties}
    >
      {children}
    </div>
  );
}

/** Animated number counter that starts when visible. */
export function Counter({ value, duration = 1600 }: { value: number; duration?: number }) {
  const ref = useRef<HTMLSpanElement>(null);
  const [n, setN] = useState(0);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    let raf = 0;
    const io = new IntersectionObserver(
      ([entry]) => {
        if (!entry.isIntersecting) return;
        io.disconnect();
        const start = performance.now();
        const tick = (now: number) => {
          const p = Math.min((now - start) / duration, 1);
          const eased = 1 - Math.pow(1 - p, 3);
          setN(Math.round(value * eased));
          if (p < 1) raf = requestAnimationFrame(tick);
        };
        raf = requestAnimationFrame(tick);
      },
      { threshold: 0.4 },
    );
    io.observe(el);
    return () => {
      io.disconnect();
      cancelAnimationFrame(raf);
    };
  }, [value, duration]);

  return <span ref={ref}>{n}</span>;
}

/** Rotating typewriter text. */
export function Typewriter({ phrases }: { phrases: string[] }) {
  const [index, setIndex] = useState(0);
  const [text, setText] = useState("");
  const [deleting, setDeleting] = useState(false);
  const key = phrases.join("|");

  useEffect(() => {
    setIndex(0);
    setText("");
    setDeleting(false);
  }, [key]);

  useEffect(() => {
    if (!phrases.length) return;
    const full = phrases[index % phrases.length];
    let delay = deleting ? 28 : 65;
    if (!deleting && text === full) delay = 1700;
    if (deleting && text === "") delay = 350;
    const id = setTimeout(() => {
      if (!deleting && text === full) {
        setDeleting(true);
      } else if (deleting && text === "") {
        setDeleting(false);
        setIndex((i) => (i + 1) % phrases.length);
      } else {
        setText(deleting ? full.slice(0, text.length - 1) : full.slice(0, text.length + 1));
      }
    }, delay);
    return () => clearTimeout(id);
  }, [text, deleting, index, phrases]);

  return (
    <span className="text-gold-400">
      {text}
      <span className="anim-blink ml-0.5 inline-block w-[3px] translate-y-[2px] bg-gold-400 align-baseline">&nbsp;</span>
    </span>
  );
}

/** Global effects: cursor spotlight on cards + scroll progress bar. */
export function GlobalFx() {
  const bar = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const onMove = (e: MouseEvent) => {
      const target = (e.target as HTMLElement | null)?.closest<HTMLElement>(".spotlight");
      if (!target) return;
      const rect = target.getBoundingClientRect();
      target.style.setProperty("--mx", `${e.clientX - rect.left}px`);
      target.style.setProperty("--my", `${e.clientY - rect.top}px`);
    };
    let raf = 0;
    const onScroll = () => {
      cancelAnimationFrame(raf);
      raf = requestAnimationFrame(() => {
        const h = document.documentElement;
        const max = h.scrollHeight - h.clientHeight;
        if (bar.current) bar.current.style.transform = `scaleX(${max > 0 ? h.scrollTop / max : 0})`;
      });
    };
    document.addEventListener("mousemove", onMove, { passive: true });
    window.addEventListener("scroll", onScroll, { passive: true });
    onScroll();
    return () => {
      document.removeEventListener("mousemove", onMove);
      window.removeEventListener("scroll", onScroll);
      cancelAnimationFrame(raf);
    };
  }, []);

  return (
    <div className="pointer-events-none fixed left-0 top-0 z-[60] h-[3px] w-full">
      <div
        ref={bar}
        className="h-full origin-left bg-gradient-to-r from-gold-500 via-gold-300 to-white"
        style={{ transform: "scaleX(0)" }}
      />
    </div>
  );
}
"use client";

import Link from "next/link";
import { ArrowRight, BadgeCheck, Bot, Link2, Mic, Trophy } from "lucide-react";
import { useI18n } from "./I18nProvider";
import { Rings } from "./Logo";
import { Counter, Typewriter } from "./fx";
import { openAssistant } from "./Navbar";
import type { StatsDTO } from "@/lib/types";

const TECH = [
  "Solidity", "TypeScript", "Next.js", "Python", "Rust", "Flutter", "Golang", "PostgreSQL", "React",
  "PyTorch", "Kubernetes", "Hardhat", "IPFS", "Figma", "Docker", "Tailwind", "Web3", "Machine Learning",
];

export function Hero({ stats }: { stats: StatsDTO }) {
  const { t } = useI18n();
  const phrases = t("hero.typed").split("|");

  const statItems = [
    { label: t("stats.developers"), value: stats.developers },
    { label: t("stats.hackathons"), value: stats.hackathons },
    { label: t("stats.projects"), value: stats.projects },
    { label: t("stats.credentials"), value: stats.credentials },
  ];

  return (
    <section className="relative overflow-hidden">
      {/* ambient blobs */}
      <div className="anim-float-slow pointer-events-none absolute -left-32 top-20 h-96 w-96 rounded-full bg-sky-500/15 blur-3xl" />
      <div className="anim-float-slow pointer-events-none absolute -right-24 top-0 h-[28rem] w-[28rem] rounded-full bg-gold-400/15 blur-3xl" style={{ animationDelay: "-7s" }} />

      <div className="relative mx-auto grid max-w-7xl items-center gap-12 px-4 pb-10 pt-12 sm:px-6 lg:grid-cols-[1.1fr_0.9fr] lg:px-8 lg:pb-16 lg:pt-20">
        <div>
          <p className="anim-rise mb-6 inline-flex items-center gap-2 rounded-full border border-gold-400/30 bg-gold-400/10 px-4 py-1.5 text-xs font-bold uppercase tracking-wider text-gold-300 sm:text-sm">
            <span className="relative flex h-2 w-2">
              <span className="absolute inline-flex h-full w-full animate-ping rounded-full bg-gold-400 opacity-75" />
              <span className="relative inline-flex h-2 w-2 rounded-full bg-gold-400" />
            </span>
            {t("hero.badge")}
          </p>
          <h1 className="anim-rise font-display text-4xl font-extrabold leading-[1.05] tracking-tight text-white sm:text-6xl lg:text-7xl" style={{ animationDelay: "80ms" }}>
            {t("hero.title1")}
            <br />
            <span className="text-gradient">{t("hero.title2")}</span>
          </h1>
          <p className="anim-rise mt-6 h-8 text-xl font-semibold text-white sm:text-2xl" style={{ animationDelay: "160ms" }}>
            <Typewriter phrases={phrases} />
          </p>
          <p className="anim-rise mt-4 max-w-xl text-base leading-relaxed text-slate-300 sm:text-lg" style={{ animationDelay: "220ms" }}>
            {t("hero.subtitle")}
          </p>
          <div className="anim-rise mt-8 flex flex-wrap items-center gap-3" style={{ animationDelay: "300ms" }}>
            <Link href="/hackathons" className="btn btn-primary">
              {t("hero.cta1")} <ArrowRight size={18} className="arrow" />
            </Link>
            <Link href="/developers/new" className="btn btn-ghost">
              {t("hero.cta2")}
            </Link>
            <button type="button" onClick={() => openAssistant("voice")} className="btn btn-ghost">
              <Mic size={17} className="text-gold-400" /> {t("hero.cta3")}
            </button>
          </div>
        </div>

        {/* visual */}
        <div className="anim-rise relative mx-auto aspect-square w-full max-w-[460px]" style={{ animationDelay: "200ms" }}>
          <div className="absolute inset-[6%] rounded-full bg-gold-400/10 blur-2xl" />
          <div className="anim-spin-slow absolute inset-0 rounded-full border border-dashed border-white/20">
            <span className="absolute left-1/2 top-0 h-3 w-3 -translate-x-1/2 -translate-y-1/2 rounded-full bg-gold-400 shadow-[0_0_20px_4px_rgba(255,212,62,0.7)]" />
            <span className="absolute bottom-[14%] right-[6%] h-2 w-2 rounded-full bg-white shadow-[0_0_14px_3px_rgba(255,255,255,0.6)]" />
          </div>
          <div className="anim-spin-rev absolute inset-[11%] rounded-full border border-white/12">
            <span className="absolute left-0 top-1/2 h-2.5 w-2.5 -translate-x-1/2 -translate-y-1/2 rounded-full bg-sky-300 shadow-[0_0_16px_3px_rgba(125,211,252,0.7)]" />
          </div>
          <div className="absolute inset-[18%] grid place-items-center rounded-[2.5rem] border border-white/10 bg-navy-800/80 shadow-[0_30px_80px_-20px_rgba(0,0,0,0.8)] backdrop-blur">
            <Rings animated className="w-[82%]" />
          </div>

          <div className="anim-float absolute left-0 top-[8%] flex items-center gap-2 rounded-2xl border border-emerald-300/30 bg-navy-900/85 px-3.5 py-2 text-xs font-semibold text-emerald-200 shadow-xl backdrop-blur" style={{ ["--r" as string]: "-3deg" }}>
            <BadgeCheck size={16} /> {t("hero.chipVerified")}
          </div>
          <div className="anim-float absolute -right-2 top-[30%] flex items-center gap-2 rounded-2xl border border-gold-400/40 bg-navy-900/85 px-3.5 py-2 text-xs font-semibold text-gold-300 shadow-xl backdrop-blur" style={{ animationDelay: "-2s", ["--r" as string]: "3deg" }}>
            <Trophy size={16} /> {t("hero.chipWinner")}
          </div>
          <div className="anim-float absolute bottom-[4%] left-[8%] flex items-center gap-2 rounded-2xl border border-white/15 bg-navy-900/85 px-3.5 py-2 text-xs shadow-xl backdrop-blur" style={{ animationDelay: "-4s", ["--r" as string]: "-2deg" }}>
            <Link2 size={15} className="text-sky-300" />
            <span className="font-mono text-slate-200">0x7f3a…c91e</span>
            <span className="text-slate-400">· {t("hero.chipTx")}</span>
          </div>
          <div className="anim-float absolute -bottom-1 right-[6%] grid h-12 w-12 place-items-center rounded-2xl bg-gradient-to-br from-gold-300 to-gold-500 text-navy-800 shadow-xl" style={{ animationDelay: "-1s" }}>
            <Bot size={24} />
          </div>
        </div>
      </div>

      {/* stats */}
      <div className="relative mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
        <div className="card grid grid-cols-2 gap-px overflow-hidden bg-white/10 md:grid-cols-4">
          {statItems.map((s) => (
            <div key={s.label} className="bg-navy-900/80 px-6 py-6 text-center transition-colors hover:bg-navy-800/90">
              <p className="font-display text-4xl font-extrabold text-gold-400 sm:text-5xl">
                <Counter value={s.value} />
                <span className="text-white/60">+</span>
              </p>
              <p className="mt-1 text-sm text-slate-300">{s.label}</p>
            </div>
          ))}
        </div>
      </div>

      {/* tech marquee */}
      <div className="marquee-wrap relative mt-10 overflow-hidden py-4 [mask-image:linear-gradient(90deg,transparent,black_10%,black_90%,transparent)]">
        <div className="anim-marquee flex w-max gap-3">
          {[...TECH, ...TECH].map((tech, i) => (
            <span key={i} className="chip !px-4 !py-2 !text-sm">
              <span className="h-1.5 w-1.5 rounded-full bg-gold-400" /> {tech}
            </span>
          ))}
        </div>
      </div>
    </section>
  );
}
"use client";

import { useRouter } from "next/navigation";
import {
  createContext,
  useCallback,
  useContext,
  useMemo,
  useState,
  type ReactNode,
} from "react";
import { translate, type DictKey } from "@/lib/dict";
import { pickText, type I18nText, type Locale } from "@/lib/types";

interface I18nContextValue {
  locale: Locale;
  setLocale: (l: Locale) => void;
  t: (key: DictKey, vars?: Record<string, string | number>) => string;
  pick: (text: I18nText | null | undefined) => string;
}

const I18nContext = createContext<I18nContextValue | null>(null);

export function I18nProvider({
  initialLocale,
  children,
}: {
  initialLocale: Locale;
  children: ReactNode;
}) {
  const [locale, setLocaleState] = useState<Locale>(initialLocale);
  const router = useRouter();

  const setLocale = useCallback(
    (l: Locale) => {
      document.cookie = `hc_locale=${l}; path=/; max-age=31536000; samesite=lax`;
      document.documentElement.lang = l;
      setLocaleState(l);
      router.refresh();
    },
    [router],
  );

  const value = useMemo<I18nContextValue>(
    () => ({
      locale,
      setLocale,
      t: (key, vars) => translate(locale, key, vars),
      pick: (text) => pickText(text, locale),
    }),
    [locale, setLocale],
  );

  return <I18nContext.Provider value={value}>{children}</I18nContext.Provider>;
}

export function useI18n(): I18nContextValue {
  const ctx = useContext(I18nContext);
  if (!ctx) throw new Error("useI18n must be used inside I18nProvider");
  return ctx;
}
"use client";

import { useId } from "react";

/** Two interlocked rings — the HackChain mark (matches the app icon). */
export function Rings({
  className,
  animated = false,
}: {
  className?: string;
  animated?: boolean;
}) {
  const id = useId().replace(/:/g, "");
  return (
    <svg
      viewBox="50 150 410 215"
      className={className}
      fill="none"
      aria-hidden="true"
      role="img"
    >
      <defs>
        <clipPath id={`lower-${id}`}>
          <rect x="190" y="262" width="140" height="120" />
        </clipPath>
      </defs>
      <g className={animated ? "ring-glow" : undefined}>
        <g className={animated ? "ring-l" : undefined}>
          <rect
            className={animated ? "ring-draw" : undefined}
            x="87"
            y="187"
            width="201"
            height="139"
            rx="69.5"
            stroke="#ffd43e"
            strokeWidth="38"
          />
        </g>
        <g className={animated ? "ring-r" : undefined}>
          <rect
            className={animated ? "ring-draw" : undefined}
            x="226"
            y="187"
            width="201"
            height="139"
            rx="69.5"
            stroke="#ffffff"
            strokeWidth="38"
            style={animated ? { animationDelay: "0.5s" } : undefined}
          />
        </g>
        {/* yellow ring passes over the white one at the bottom crossing */}
        <g clipPath={`url(#lower-${id})`}>
          <g className={animated ? "ring-l" : undefined}>
            <rect x="87" y="187" width="201" height="139" rx="69.5" stroke="#ffd43e" strokeWidth="38" />
          </g>
        </g>
      </g>
    </svg>
  );
}

/** App-icon style mark: navy rounded square + rings. */
export function LogoMark({ size = 40, className }: { size?: number; className?: string }) {
  const id = useId().replace(/:/g, "");
  return (
    <svg
      width={size}
      height={size}
      viewBox="0 0 512 512"
      className={className}
      aria-hidden="true"
    >
      <defs>
        <clipPath id={`m-${id}`}>
          <rect x="190" y="262" width="140" height="120" />
        </clipPath>
      </defs>
      <rect width="512" height="512" rx="112" fill="#13243b" />
      <rect x="87" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#ffd43e" strokeWidth="38" />
      <rect x="226" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#fff" strokeWidth="38" />
      <g clipPath={`url(#m-${id})`}>
        <rect x="87" y="187" width="201" height="139" rx="69.5" fill="none" stroke="#ffd43e" strokeWidth="38" />
      </g>
    </svg>
  );
}

export function Wordmark({ size = 36 }: { size?: number }) {
  return (
    <span className="group inline-flex items-center gap-2.5">
      <span className="transition-transform duration-500 group-hover:rotate-[-8deg] group-hover:scale-110">
        <LogoMark size={size} className="rounded-[28%] shadow-[0_0_0_1px_rgba(255,255,255,0.1),0_8px_24px_-8px_rgba(255,212,62,0.5)]" />
      </span>
      <span className="font-display text-xl font-extrabold tracking-tight text-white">
        Hack<span className="text-gold-400">Chain</span>
      </span>
    </span>
  );
}
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { useEffect, useState } from "react";
import { Bot, Menu, Plus, X } from "lucide-react";
import { useI18n } from "./I18nProvider";
import { Wordmark } from "./Logo";
import { LOCALE_META } from "@/lib/dict";
import { LOCALES } from "@/lib/types";

export function openAssistant(tab: "chat" | "voice" = "chat") {
  window.dispatchEvent(new CustomEvent("hackchain:assistant", { detail: { tab } }));
}

export function LangSwitch({ className = "" }: { className?: string }) {
  const { locale, setLocale } = useI18n();
  const idx = LOCALES.indexOf(locale);
  return (
    <div
      role="group"
      aria-label="Language"
      className={`relative inline-grid grid-cols-3 rounded-full border border-white/12 bg-white/5 p-1 ${className}`}
    >
      <span
        className="absolute bottom-1 left-1 top-1 w-[calc((100%-0.5rem)/3)] rounded-full bg-gradient-to-br from-gold-400 to-gold-500 shadow-[0_4px_14px_-2px_rgba(255,212,62,0.6)] transition-transform duration-300 ease-out"
        style={{ transform: `translateX(${idx * 100}%)` }}
      />
      {LOCALES.map((l) => (
        <button
          key={l}
          type="button"
          onClick={() => setLocale(l)}
          title={LOCALE_META[l].label}
          aria-pressed={l === locale}
          className={`relative z-10 cursor-pointer px-3 py-1 text-xs font-bold tracking-wide transition-colors ${
            l === locale ? "text-navy-800" : "text-slate-300 hover:text-white"
          }`}
        >
          {LOCALE_META[l].short}
        </button>
      ))}
    </div>
  );
}

export function Navbar() {
  const { t } = useI18n();
  const pathname = usePathname();
  const [scrolled, setScrolled] = useState(false);
  const [open, setOpen] = useState(false);

  useEffect(() => {
    const onScroll = () => setScrolled(window.scrollY > 12);
    onScroll();
    window.addEventListener("scroll", onScroll, { passive: true });
    return () => window.removeEventListener("scroll", onScroll);
  }, []);

  useEffect(() => {
    setOpen(false);
  }, [pathname]);

  const links = [
    { href: "/hackathons", label: t("nav.hackathons") },
    { href: "/projects", label: t("nav.projects") },
    { href: "/developers", label: t("nav.developers") },
    { href: "/verify", label: t("nav.verify") },
  ];

  const isActive = (href: string) => pathname === href || pathname.startsWith(href + "/");

  return (
    <header
      className={`fixed inset-x-0 top-0 z-50 transition-all duration-300 ${
        scrolled || open
          ? "border-b border-white/10 bg-navy-900/80 shadow-[0_10px_30px_-15px_rgba(0,0,0,0.7)] backdrop-blur-xl"
          : "border-b border-transparent bg-transparent"
      }`}
    >
      <div className="mx-auto flex h-16 max-w-7xl items-center justify-between gap-4 px-4 sm:px-6 lg:px-8">
        <Link href="/" aria-label="HackChain">
          <Wordmark />
        </Link>

        <nav className="hidden items-center gap-1 lg:flex">
          {links.map((l) => (
            <Link
              key={l.href}
              href={l.href}
              className={`group relative rounded-full px-4 py-2 text-sm font-medium transition-colors ${
                isActive(l.href) ? "text-gold-400" : "text-slate-300 hover:text-white"
              }`}
            >
              {l.label}
              <span
                className={`absolute inset-x-4 -bottom-0.5 h-0.5 origin-left rounded-full bg-gold-400 transition-transform duration-300 ${
                  isActive(l.href) ? "scale-x-100" : "scale-x-0 group-hover:scale-x-100"
                }`}
              />
            </Link>
          ))}
        </nav>

        <div className="flex items-center gap-2 sm:gap-3">
          <LangSwitch className="hidden sm:inline-grid" />
          <button
            type="button"
            onClick={() => openAssistant("chat")}
            className="btn btn-ghost btn-sm hidden md:inline-flex"
          >
            <Bot size={16} className="text-gold-400" />
            <span className="hidden xl:inline">{t("nav.askAi")}</span>
          </button>
          <Link href="/developers/new" className="btn btn-primary btn-sm hidden md:inline-flex">
            <Plus size={16} />
            <span className="hidden xl:inline">{t("nav.createProfile")}</span>
            <span className="xl:hidden">{t("nav.createProfile").split(" ")[0]}</span>
          </Link>
          <button
            type="button"
            aria-label={t("nav.menu")}
            onClick={() => setOpen((v) => !v)}
            className="grid h-10 w-10 cursor-pointer place-items-center rounded-full border border-white/12 bg-white/5 text-white transition hover:border-gold-400/60 lg:hidden"
          >
            {open ? <X size={20} /> : <Menu size={20} />}
          </button>
        </div>
      </div>

      {/* mobile menu */}
      <div
        className={`grid overflow-hidden transition-[grid-template-rows] duration-300 lg:hidden ${
          open ? "grid-rows-[1fr]" : "grid-rows-[0fr]"
        }`}
      >
        <div className="min-h-0 overflow-hidden">
          <div className="space-y-1 px-4 pb-5 pt-2">
            {links.map((l) => (
              <Link
                key={l.href}
                href={l.href}
                className={`block rounded-xl px-4 py-3 text-base font-medium transition ${
                  isActive(l.href) ? "bg-gold-400/10 text-gold-400" : "text-slate-200 hover:bg-white/5"
                }`}
              >
                {l.label}
              </Link>
            ))}
            <div className="flex flex-wrap items-center gap-3 pt-3">
              <LangSwitch />
              <button type="button" onClick={() => openAssistant("voice")} className="btn btn-ghost btn-sm">
                <Bot size={16} className="text-gold-400" /> {t("nav.askAi")}
              </button>
              <Link href="/developers/new" className="btn btn-primary btn-sm">
                <Plus size={16} /> {t("nav.createProfile")}
              </Link>
            </div>
          </div>
        </div>
      </div>
    </header>
  );
}
import type { ReactNode } from "react";

export function PageHeader({
  eyebrow,
  title,
  sub,
  children,
}: {
  eyebrow?: string;
  title: string;
  sub?: string;
  children?: ReactNode;
}) {
  return (
    <section className="relative overflow-hidden border-b border-white/8">
      <div className="pointer-events-none absolute -right-24 -top-24 h-72 w-72 rounded-full bg-gold-400/10 blur-3xl anim-float-slow" />
      <div className="pointer-events-none absolute -left-24 top-10 h-64 w-64 rounded-full bg-sky-500/10 blur-3xl anim-float-slow" style={{ animationDelay: "-6s" }} />
      <div className="relative mx-auto max-w-7xl px-4 pb-10 pt-14 sm:px-6 lg:px-8">
        {eyebrow && (
          <p className="anim-rise mb-3 inline-flex items-center gap-2 rounded-full border border-gold-400/30 bg-gold-400/10 px-3 py-1 text-xs font-bold uppercase tracking-wider text-gold-300">
            {eyebrow}
          </p>
        )}
        <h1 className="anim-rise font-display text-4xl font-extrabold tracking-tight text-white sm:text-5xl" style={{ animationDelay: "60ms" }}>
          {title}
        </h1>
        {sub && (
          <p className="anim-rise mt-4 max-w-2xl text-lg text-slate-300" style={{ animationDelay: "120ms" }}>
            {sub}
          </p>
        )}
        {children && (
          <div className="anim-rise mt-6" style={{ animationDelay: "180ms" }}>
            {children}
          </div>
        )}
      </div>
    </section>
  );
}

export function Container({ children, className = "" }: { children: ReactNode; className?: string }) {
  return <div className={`mx-auto max-w-7xl px-4 py-10 sm:px-6 lg:px-8 ${className}`}>{children}</div>;
}
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";

const databaseUrl = process.env.DATABASE_URL;

if (!databaseUrl) {
  throw new Error("DATABASE_URL is required");
}

const globalForDb = globalThis as typeof globalThis & {
  __arenaNextJsPostgresqlPool?: Pool;
};

export const pool =
  globalForDb.__arenaNextJsPostgresqlPool ??
  new Pool({
    connectionString: databaseUrl,
  });

if (process.env.NODE_ENV !== "production") {
  globalForDb.__arenaNextJsPostgresqlPool = pool;
}

export const db = drizzle(pool);
import {
  boolean,
  integer,
  jsonb,
  pgTable,
  serial,
  text,
  timestamp,
  unique,
} from "drizzle-orm/pg-core";

export type I18nText = { en: string; ru: string; kk: string };

/** Developer portfolio / profile */
export const developers = pgTable("developers", {
  id: serial("id").primaryKey(),
  handle: text("handle").notNull().unique(),
  name: text("name").notNull(),
  role: jsonb("role").$type<I18nText>().notNull(),
  bio: jsonb("bio").$type<I18nText>().notNull(),
  location: text("location").notNull().default(""),
  skills: text("skills").array().notNull().default([]),
  github: text("github"),
  website: text("website"),
  telegram: text("telegram"),
  wallet: text("wallet"),
  openToWork: boolean("open_to_work").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

/** Hackathons / IT tournaments */
export const hackathons = pgTable("hackathons", {
  id: serial("id").primaryKey(),
  slug: text("slug").notNull().unique(),
  title: jsonb("title").$type<I18nText>().notNull(),
  description: jsonb("description").$type<I18nText>().notNull(),
  organizer: text("organizer").notNull(),
  organizerId: integer("organizer_id").references(() => developers.id, {
    onDelete: "set null",
  }),
  format: text("format").notNull().default("online"),
  location: text("location").notNull().default(""),
  startsAt: timestamp("starts_at").notNull(),
  endsAt: timestamp("ends_at").notNull(),
  prizePool: text("prize_pool").notNull().default(""),
  maxParticipants: integer("max_participants").notNull().default(100),
  tags: text("tags").array().notNull().default([]),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

/** Projects (small and large-scale) */
export const projects = pgTable("projects", {
  id: serial("id").primaryKey(),
  slug: text("slug").notNull().unique(),
  title: jsonb("title").$type<I18nText>().notNull(),
  summary: jsonb("summary").$type<I18nText>().notNull(),
  description: jsonb("description").$type<I18nText>().notNull(),
  scale: text("scale").notNull().default("small"),
  category: text("category").notNull().default("web"),
  tags: text("tags").array().notNull().default([]),
  repoUrl: text("repo_url"),
  demoUrl: text("demo_url"),
  likes: integer("likes").notNull().default(0),
  ownerId: integer("owner_id").references(() => developers.id, {
    onDelete: "set null",
  }),
  hackathonId: integer("hackathon_id").references(() => hackathons.id, {
    onDelete: "set null",
  }),
  verified: boolean("verified").notNull().default(false),
  txHash: text("tx_hash"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

/** Participant registrations for tournaments */
export const registrations = pgTable(
  "registrations",
  {
    id: serial("id").primaryKey(),
    hackathonId: integer("hackathon_id")
      .notNull()
      .references(() => hackathons.id, { onDelete: "cascade" }),
    developerId: integer("developer_id").references(() => developers.id, {
      onDelete: "set null",
    }),
    name: text("name").notNull(),
    email: text("email").notNull(),
    teamName: text("team_name").notNull().default(""),
    experience: text("experience").notNull().default("middle"),
    createdAt: timestamp("created_at").notNull().defaultNow(),
  },
  (t) => [unique("registrations_hackathon_email_uq").on(t.hackathonId, t.email)],
);

/** Organizer-verified achievements written to the ledger */
export const achievements = pgTable("achievements", {
  id: serial("id").primaryKey(),
  developerId: integer("developer_id")
    .notNull()
    .references(() => developers.id, { onDelete: "cascade" }),
  hackathonId: integer("hackathon_id").references(() => hackathons.id, {
    onDelete: "set null",
  }),
  kind: text("kind").notNull().default("participation"),
  title: jsonb("title").$type<I18nText>().notNull(),
  issuer: text("issuer").notNull(),
  place: integer("place"),
  txHash: text("tx_hash").notNull().unique(),
  verified: boolean("verified").notNull().default(true),
  awardedAt: timestamp("awarded_at").notNull().defaultNow(),
});
import { sql } from "drizzle-orm";
import { db } from "./index";
import {
  achievements,
  developers,
  hackathons,
  projects,
  registrations,
  type I18nText,
} from "./schema";
import { fakeTxHash } from "../lib/utils";

const L = (en: string, ru: string, kk: string): I18nText => ({ en, ru, kk });
const N = (name: string): I18nText => ({ en: name, ru: name, kk: name });

const DAY = 86_400_000;
const at = (offsetDays: number, hour = 10) => {
  const d = new Date(Date.now() + offsetDays * DAY);
  d.setUTCHours(hour, 0, 0, 0);
  return d;
};

let seedPromise: Promise<void> | null = null;

export function ensureSeeded(): Promise<void> {
  if (!seedPromise) {
    seedPromise = runSeed().catch((err) => {
      seedPromise = null;
      throw err;
    });
  }
  return seedPromise;
}

async function runSeed() {
  await db.transaction(async (tx) => {
    await tx.execute(sql`select pg_advisory_xact_lock(748201)`);
    const existing = await tx.execute(sql`select count(*)::int as n from developers`);
    const n = Number((existing.rows[0] as { n: number } | undefined)?.n ?? 0);
    if (n > 0) return;

    // ---------- Developers ----------
    const devRows = await tx
      .insert(developers)
      .values([
        {
          handle: "aruzhan",
          name: "Aruzhan Sadykova",
          role: L("Full-stack Engineer", "Full-stack разработчик", "Full-stack әзірлеуші"),
          bio: L(
            "I build product-minded web apps and mentor first-time hackathon teams. Three-time finalist who loves turning weekend ideas into real products.",
            "Создаю продуктовые веб-приложения и менторю команды на их первом хакатоне. Трёхкратный финалист, люблю превращать идеи выходного дня в настоящие продукты.",
            "Өнімге бағытталған веб-қосымшалар жасаймын және алғаш хакатонға қатысатын командаларға тәлімгерлік етемін. Үш мәрте финалист, демалыс күнгі идеяларды нақты өнімге айналдырғанды жақсы көремін.",
          ),
          location: "Almaty, Kazakhstan",
          skills: ["TypeScript", "React", "Next.js", "PostgreSQL", "Node.js"],
          github: "https://github.com/aruzhan-dev",
          telegram: "@aruzhan_dev",
          wallet: "0x4aF1c2e9B7d3a05C18E6f2a9D4b7C3e1F0a8b256",
        },
        {
          handle: "daniyar",
          name: "Daniyar Omarov",
          role: L("Smart Contract Developer", "Разработчик смарт-контрактов", "Смарт-контракт әзірлеушісі"),
          bio: L(
            "Solidity engineer focused on identity and reputation protocols. Winner of Web3 Builders Summit and a firm believer that credentials should belong to people, not platforms.",
            "Solidity-инженер, работаю над протоколами идентичности и репутации. Победитель Web3 Builders Summit, уверен, что достижения должны принадлежать людям, а не платформам.",
            "Идентификация мен бедел протоколдарына маманданған Solidity-инженермін. Web3 Builders Summit жеңімпазы, марапаттар платформаға емес, адамдарға тиесілі деп сенемін.",
          ),
          location: "Astana, Kazakhstan",
          skills: ["Solidity", "Hardhat", "Ethers.js", "IPFS", "Rust"],
          github: "https://github.com/daniyar-sol",
          telegram: "@daniyar_sol",
          wallet: "0x9Bc3D1a7E5f40A2c86b1E37d90F4a5C2e8D61f03",
        },
        {
          handle: "aigerim",
          name: "Aigerim Nurlanova",
          role: L("Machine Learning Engineer", "ML-инженер", "ML-инженер"),
          bio: L(
            "I teach machines to understand Kazakh. Author of SteppeGPT and a regular on AI hackathon podiums — I care about language tech for under-represented languages.",
            "Учу машины понимать казахский язык. Автор SteppeGPT и частый гость пьедесталов AI-хакатонов — занимаюсь языковыми технологиями для малых языков.",
            "Машиналарды қазақ тілін түсінуге үйретемін. SteppeGPT авторы және AI-хакатондар тұғырының тұрақты қонағы — аз қолданылатын тілдерге арналған тіл технологияларымен айналысамын.",
          ),
          location: "Almaty, Kazakhstan",
          skills: ["Python", "PyTorch", "LLM", "FastAPI", "Machine Learning"],
          github: "https://github.com/aigerim-ml",
          website: "https://aigerim.example.com",
          telegram: "@aigerim_ml",
          wallet: "0x71E0aC95d2B8F3c4a6D1e57B09c3F8a24D6e1b9C",
        },
        {
          handle: "timur",
          name: "Timur Bekov",
          role: L("Mobile Developer", "Мобильный разработчик", "Мобильді әзірлеуші"),
          bio: L(
            "Flutter developer who ships polished apps in 48 hours. I love tiny delightful tools that people use every day.",
            "Flutter-разработчик, который выпускает отполированные приложения за 48 часов. Люблю небольшие приятные инструменты, которыми пользуются каждый день.",
            "48 сағатта жылтыратып қосымша шығаратын Flutter-әзірлеушімін. Адамдар күнде қолданатын шағын әрі ыңғайлы құралдарды жақсы көремін.",
          ),
          location: "Shymkent, Kazakhstan",
          skills: ["Flutter", "Dart", "Kotlin", "Firebase", "Figma"],
          github: "https://github.com/timur-mobile",
          telegram: "@timur_mobile",
        },
        {
          handle: "alikhan",
          name: "Alikhan Serikov",
          role: L("Backend Engineer", "Backend-инженер", "Backend-инженер"),
          bio: L(
            "Backend and infrastructure engineer. I build fast, boring and reliable systems in Go — payments, queues and everything that must not go down.",
            "Backend- и инфраструктурный инженер. Строю быстрые, скучные и надёжные системы на Go — платежи, очереди и всё, что не имеет права падать.",
            "Backend және инфрақұрылым инженерімін. Go тілінде жылдам, қарапайым әрі сенімді жүйелер жасаймын — төлемдер, кезектер және құламауы тиіс барлық нәрсе.",
          ),
          location: "Karaganda, Kazakhstan",
          skills: ["Golang", "Docker", "Kubernetes", "gRPC", "Redis"],
          github: "https://github.com/alikhan-go",
          telegram: "@alikhan_go",
          wallet: "0x5D27f8Ae10c9B34e6a7F2d83C1b05E9a4F6c7D18",
        },
        {
          handle: "madina",
          name: "Madina Zhaksylykova",
          role: L("UI/UX Designer & Frontend", "UI/UX дизайнер и Frontend", "UI/UX дизайнер және Frontend"),
          bio: L(
            "Designer who codes. I make interfaces feel alive with motion and clear storytelling, and I've won best-design awards at three events.",
            "Дизайнер, который пишет код. Оживляю интерфейсы анимацией и понятным сторителлингом, получала награды за лучший дизайн на трёх событиях.",
            "Код жазатын дизайнермін. Интерфейстерді анимациямен және түсінікті баяндаумен жандандырамын, үш іс-шарада үздік дизайн марапатын алғанмын.",
          ),
          location: "Almaty, Kazakhstan",
          skills: ["Figma", "React", "Tailwind CSS", "Animation", "Design Systems"],
          github: "https://github.com/madina-ui",
          website: "https://madina.example.com",
          telegram: "@madina_ui",
        },
        {
          handle: "nurzhan",
          name: "Nurzhan Abenov",
          role: L("Security Engineer", "Инженер по безопасности", "Қауіпсіздік инженері"),
          bio: L(
            "Security researcher and CTF player. I audit smart contracts and web apps, and I run workshops on practical offensive security.",
            "Исследователь безопасности и CTF-игрок. Провожу аудит смарт-контрактов и веб-приложений, веду практические воркшопы по наступательной безопасности.",
            "Қауіпсіздік зерттеушісі және CTF ойыншысы. Смарт-контракттар мен веб-қосымшаларға аудит жасаймын, шабуылдық қауіпсіздік бойынша практикалық воркшоптар өткіземін.",
          ),
          location: "Astana, Kazakhstan",
          skills: ["Cybersecurity", "Pentesting", "Linux", "Python", "Auditing"],
          github: "https://github.com/nurzhan-sec",
          telegram: "@nurzhan_sec",
          wallet: "0x3C8e16Fd0A92b7E54c1D6a38F0b2E9d75A4c1B60",
        },
        {
          handle: "sofia",
          name: "Sofia Petrova",
          role: L("Data Scientist", "Data Scientist", "Data Scientist"),
          bio: L(
            "Data scientist working on agriculture and climate analytics across Central Asia. I turn messy sensor data into decisions farmers can act on.",
            "Data scientist, занимаюсь аналитикой в сельском хозяйстве и климате Центральной Азии. Превращаю хаотичные данные датчиков в решения, на основе которых фермеры могут действовать.",
            "Орталық Азиядағы ауыл шаруашылығы мен климат аналитикасымен айналысатын data scientist-пін. Датчиктердің ретсіз деректерін фермерлер қолдана алатын шешімдерге айналдырамын.",
          ),
          location: "Bishkek, Kyrgyzstan",
          skills: ["Python", "SQL", "Pandas", "Tableau", "Analytics"],
          github: "https://github.com/sofia-data",
          telegram: "@sofia_data",
        },
      ])
      .returning({ id: developers.id, handle: developers.handle, name: developers.name });
    const dev = Object.fromEntries(devRows.map((d) => [d.handle, d.id])) as Record<string, number>;

    // ---------- Hackathons ----------
    const hackRows = await tx
      .insert(hackathons)
      .values([
        {
          slug: "hackchain-genesis",
          title: N("HackChain Genesis Hackathon"),
          description: L(
            "The flagship HackChain event. Build the future of verifiable developer reputation: identity, credentials, DeFi, AI agents and developer tools. Three tracks, expert mentors, and every winner gets an on-chain credential.",
            "Флагманское событие HackChain. Создавайте будущее проверяемой репутации разработчиков: идентичность, награды, DeFi, ИИ-агенты и инструменты для разработчиков. Три трека, менторы-эксперты, а каждый победитель получает награду в блокчейне.",
            "HackChain-нің басты іс-шарасы. Әзірлеушілердің тексерілетін беделінің болашағын құрыңыз: сәйкестендіру, марапаттар, DeFi, ЖИ-агенттер және әзірлеуші құралдары. Үш трек, сарапшы тәлімгерлер, ал әр жеңімпаз блокчейнде марапат алады.",
          ),
          organizer: "HackChain Foundation",
          format: "hybrid",
          location: "Almaty + Online",
          startsAt: at(21, 5),
          endsAt: at(23, 15),
          prizePool: "$25,000",
          maxParticipants: 300,
          tags: ["Web3", "AI", "DeFi", "Identity"],
        },
        {
          slug: "steppe-ai-for-good",
          title: N("Steppe AI for Good"),
          description: L(
            "A 72-hour online hackathon to build AI solutions for education, healthcare and agriculture in Central Asia. Teams of up to 5, datasets and GPU credits provided.",
            "72-часовой онлайн-хакатон по созданию ИИ-решений для образования, здравоохранения и сельского хозяйства Центральной Азии. Команды до 5 человек, датасеты и GPU-кредиты предоставляются.",
            "Орталық Азиядағы білім, денсаулық сақтау және ауыл шаруашылығына арналған ЖИ-шешімдер жасайтын 72 сағаттық онлайн-хакатон. Команда 5 адамға дейін, датасеттер мен GPU-кредиттер беріледі.",
          ),
          organizer: "Steppe AI Lab",
          format: "online",
          location: "Online",
          startsAt: at(-1, 4),
          endsAt: at(2, 16),
          prizePool: "$10,000 + GPU credits",
          maxParticipants: 200,
          tags: ["AI", "Social impact", "Data"],
        },
        {
          slug: "astana-code-cup",
          title: N("Astana Code Cup 2026"),
          description: L(
            "An in-person algorithmic and backend contest in the capital. Solve real-world engineering problems in teams of three and compete for the Cup.",
            "Очный алгоритмический и backend-турнир в столице. Решайте реальные инженерные задачи в командах по три человека и боритесь за Кубок.",
            "Астанадағы алгоритмдік және backend бойынша офлайн турнир. Үш адамнан тұратын командада нақты инженерлік есептерді шешіп, Кубок үшін күресіңіз.",
          ),
          organizer: "Astana Hub Community",
          format: "offline",
          location: "Astana, Kazakhstan",
          startsAt: at(50, 4),
          endsAt: at(51, 14),
          prizePool: "$15,000",
          maxParticipants: 120,
          tags: ["Algorithms", "Backend", "Teams"],
        },
        {
          slug: "mobile-sprint-48",
          title: N("Mobile Sprint 48h"),
          description: L(
            "Ship a mobile app in 48 hours. Flutter, Kotlin, Swift — any stack goes. Judged on UX, polish and real-world usefulness.",
            "Выпустите мобильное приложение за 48 часов. Flutter, Kotlin, Swift — подойдёт любой стек. Оцениваются UX, проработка и практическая польза.",
            "48 сағатта мобильді қосымша шығарыңыз. Flutter, Kotlin, Swift — кез келген стек болады. UX, әрлеу және нақты пайдасы бағаланады.",
          ),
          organizer: "Mobile Devs KZ",
          format: "online",
          location: "Online",
          startsAt: at(75, 6),
          endsAt: at(77, 6),
          prizePool: "$5,000",
          maxParticipants: 150,
          tags: ["Mobile", "Flutter", "UX"],
        },
        {
          slug: "cybershield-ctf",
          title: N("CyberShield CTF Arena"),
          description: L(
            "A capture-the-flag tournament with web, crypto, reverse engineering and smart contract challenges. Finished with 40+ teams from 6 countries.",
            "CTF-турнир с заданиями по web, криптографии, реверс-инжинирингу и смарт-контрактам. Завершился с участием более 40 команд из 6 стран.",
            "Web, криптография, реверс-инжиниринг және смарт-контракт тапсырмалары бар CTF-турнир. 6 елден 40-тан астам команда қатысып аяқталды.",
          ),
          organizer: "CyberShield KZ",
          format: "hybrid",
          location: "Astana + Online",
          startsAt: at(-47, 5),
          endsAt: at(-45, 17),
          prizePool: "$8,000",
          maxParticipants: 160,
          tags: ["Security", "CTF", "Smart contracts"],
        },
        {
          slug: "web3-builders-summit",
          title: N("Web3 Builders Summit Hackathon"),
          description: L(
            "Two days of building on-chain products — DeFi, identity, NFTs and infrastructure. Over 60 projects were submitted and verified on-chain.",
            "Два дня создания onchain-продуктов — DeFi, идентичность, NFT и инфраструктура. Подано более 60 проектов, все результаты подтверждены в блокчейне.",
            "Екі күн бойы on-chain өнімдер жасау — DeFi, сәйкестендіру, NFT және инфрақұрылым. 60-тан астам жоба жіберілді, нәтижелер блокчейнде расталды.",
          ),
          organizer: "Web3 Kazakhstan",
          format: "offline",
          location: "Almaty, Kazakhstan",
          startsAt: at(-112, 4),
          endsAt: at(-110, 15),
          prizePool: "$20,000",
          maxParticipants: 250,
          tags: ["Web3", "DeFi", "NFT"],
        },
        {
          slug: "almaty-data-jam",
          title: N("Almaty Data Jam"),
          description: L(
            "A data science marathon on open climate and agriculture datasets. Prototype, visualize and present insights to a jury of researchers.",
            "Марафон по data science на открытых климатических и сельскохозяйственных данных. Создавайте прототипы, визуализируйте и представляйте выводы жюри исследователей.",
            "Климат пен ауыл шаруашылығының ашық деректеріндегі data science марафоны. Прототип жасаңыз, визуализация құрыңыз және зерттеушілер қазылар алқасына қорытындыны таныстырыңыз.",
          ),
          organizer: "Almaty Data Community",
          format: "offline",
          location: "Almaty, Kazakhstan",
          startsAt: at(-201, 4),
          endsAt: at(-200, 16),
          prizePool: "$6,000",
          maxParticipants: 100,
          tags: ["Data", "AI", "Climate"],
        },
      ])
      .returning({ id: hackathons.id, slug: hackathons.slug, title: hackathons.title });
    const hack = Object.fromEntries(hackRows.map((h) => [h.slug, h.id])) as Record<string, number>;

    // ---------- Projects ----------
    type P = {
      slug: string;
      title: string;
      summary: I18nText;
      description: I18nText;
      scale: "large" | "small";
      category: string;
      tags: string[];
      repo?: string;
      demo?: string;
      likes: number;
      owner: string;
      hack?: string;
      verified?: boolean;
    };
    const projectSeeds: P[] = [
      {
        slug: "chainfolio",
        title: "ChainFolio",
        summary: L(
          "An on-chain developer portfolio protocol that turns hackathon results into verifiable credentials.",
          "Протокол onchain-портфолио, превращающий результаты хакатонов в проверяемые награды.",
          "Хакатон нәтижелерін тексерілетін марапатқа айналдыратын on-chain портфолио протоколы.",
        ),
        description: L(
          "ChainFolio lets organizers sign results with their wallets and mints soulbound credentials for each participant. Includes a Solidity registry, an IPFS metadata layer and a Next.js dashboard for organizers. Won the Web3 Builders Summit.",
          "ChainFolio позволяет организаторам подписывать результаты кошельками и выпускает непередаваемые награды для каждого участника. Включает реестр на Solidity, слой метаданных на IPFS и панель организатора на Next.js. Победитель Web3 Builders Summit.",
          "ChainFolio ұйымдастырушыларға нәтижені әмиянымен қол қоюға мүмкіндік беріп, әр қатысушыға берілмейтін марапат шығарады. Solidity тізілімі, IPFS метадеректер қабаты және Next.js ұйымдастырушы панелі бар. Web3 Builders Summit жеңімпазы.",
        ),
        scale: "large",
        category: "web3",
        tags: ["Solidity", "IPFS", "Next.js", "Identity"],
        repo: "https://github.com/daniyar-sol/chainfolio",
        demo: "https://chainfolio.example.com",
        likes: 248,
        owner: "daniyar",
        hack: "web3-builders-summit",
        verified: true,
      },
      {
        slug: "steppegpt",
        title: "SteppeGPT",
        summary: L(
          "A Kazakh-first language model assistant fine-tuned on open Kazakh corpora.",
          "Языковая модель-ассистент с упором на казахский язык, дообученная на открытых корпусах.",
          "Ашық корпустарда қосымша оқытылған, қазақ тіліне басымдық беретін тілдік модель-көмекші.",
        ),
        description: L(
          "SteppeGPT answers questions, summarizes documents and translates between Kazakh, Russian and English. The training pipeline, evaluation set and a FastAPI inference server are open source.",
          "SteppeGPT отвечает на вопросы, суммирует документы и переводит между казахским, русским и английским. Конвейер обучения, оценочный набор и FastAPI-сервер инференса открыты.",
          "SteppeGPT сұрақтарға жауап береді, құжаттарды қысқартады және қазақ, орыс, ағылшын тілдері арасында аударады. Оқыту конвейері, бағалау жинағы және FastAPI инференс-сервері ашық.",
        ),
        scale: "large",
        category: "ai",
        tags: ["PyTorch", "LLM", "FastAPI", "NLP"],
        repo: "https://github.com/aigerim-ml/steppegpt",
        demo: "https://steppegpt.example.com",
        likes: 312,
        owner: "aigerim",
        hack: "almaty-data-jam",
        verified: true,
      },
      {
        slug: "qazpay-gateway",
        title: "QazPay Gateway",
        summary: L(
          "A high-throughput payment gateway with stablecoin settlement, built in Go.",
          "Высоконагруженный платёжный шлюз с расчётами в стейблкоинах, написан на Go.",
          "Stablecoin арқылы есеп айырысатын, Go тілінде жазылған жоғары жүктемелі төлем шлюзі.",
        ),
        description: L(
          "Handles thousands of transactions per second with idempotent APIs, a gRPC core and Kubernetes-native deployment. Merchants get webhooks, dashboards and instant on-chain settlement.",
          "Обрабатывает тысячи транзакций в секунду: идемпотентные API, ядро на gRPC, развёртывание в Kubernetes. Продавцы получают вебхуки, дашборды и мгновенные расчёты в блокчейне.",
          "Идемпотентті API, gRPC ядросы және Kubernetes орналастыруымен секундына мыңдаған транзакцияны өңдейді. Саудагерлер вебхуктер, дашбордтар және блокчейндегі лезде есеп айырысу алады.",
        ),
        scale: "large",
        category: "fintech",
        tags: ["Golang", "gRPC", "Kubernetes", "Stablecoin"],
        repo: "https://github.com/alikhan-go/qazpay",
        likes: 187,
        owner: "alikhan",
        hack: "web3-builders-summit",
        verified: true,
      },
      {
        slug: "agrosense",
        title: "AgroSense",
        summary: L(
          "IoT sensors and ML forecasts that help farmers decide when to water and harvest.",
          "IoT-датчики и ML-прогнозы, помогающие фермерам решать, когда поливать и собирать урожай.",
          "Фермерлерге қашан суару және егін жинау керектігін шешуге көмектесетін IoT-датчиктер мен ML-болжамдар.",
        ),
        description: L(
          "A network of low-cost soil sensors streams data to a forecasting service that predicts moisture and yield. Deployed on 12 pilot farms; includes a Tableau dashboard and an SMS alert system.",
          "Сеть недорогих датчиков почвы передаёт данные сервису прогнозирования влажности и урожайности. Развёрнуто на 12 пилотных фермах; есть дашборд в Tableau и SMS-оповещения.",
          "Арзан топырақ датчиктерінің желісі ылғалдылық пен өнімділікті болжайтын сервиске дерек жібереді. 12 пилоттық шаруашылықта қолданылады; Tableau дашборды және SMS-хабарландырулар бар.",
        ),
        scale: "large",
        category: "data",
        tags: ["IoT", "Python", "Forecasting", "Tableau"],
        repo: "https://github.com/sofia-data/agrosense",
        likes: 156,
        owner: "sofia",
        hack: "almaty-data-jam",
        verified: true,
      },
      {
        slug: "safevault-auditor",
        title: "SafeVault Auditor",
        summary: L(
          "A toolkit that scans smart contracts for common vulnerabilities and generates audit reports.",
          "Набор инструментов, который сканирует смарт-контракты на типовые уязвимости и формирует отчёты аудита.",
          "Смарт-контракттарды жиі кездесетін осалдықтарға сканерлеп, аудит есептерін жасайтын құралдар жиынтығы.",
        ),
        description: L(
          "Combines static analysis and fuzzing with a clean web report. Detects reentrancy, access-control mistakes and unchecked calls. Took first place at CyberShield CTF Arena.",
          "Объединяет статический анализ и фаззинг с понятным веб-отчётом. Находит реентранси, ошибки контроля доступа и непроверенные вызовы. Заняла первое место на CyberShield CTF Arena.",
          "Статикалық талдау мен фаззингті түсінікті веб-есеппен біріктіреді. Реентранси, қолжетімділікті бақылау қателері мен тексерілмеген шақыруларды табады. CyberShield CTF Arena-да бірінші орын алды.",
        ),
        scale: "large",
        category: "security",
        tags: ["Security", "Solidity", "Fuzzing", "Python"],
        repo: "https://github.com/nurzhan-sec/safevault",
        demo: "https://safevault.example.com",
        likes: 201,
        owner: "nurzhan",
        hack: "cybershield-ctf",
        verified: true,
      },
      {
        slug: "gas-tracker-mini",
        title: "Gas Tracker Mini",
        summary: L(
          "A tiny browser widget that shows live gas prices and the cheapest time to transact.",
          "Крошечный виджет для браузера с актуальными ценами на gas и лучшим временем для транзакции.",
          "Gas бағасы мен транзакцияға ең тиімді уақытты көрсететін шағын браузер виджеті.",
        ),
        description: L(
          "Built in a single evening: a lightweight extension that polls public RPC endpoints and charts gas history for the last 24 hours.",
          "Сделан за один вечер: лёгкое расширение, опрашивающее публичные RPC и строящее график gas за последние 24 часа.",
          "Бір кеште жасалған: көпшілік RPC-ларды сұрап, соңғы 24 сағаттағы gas графигін салатын жеңіл кеңейтім.",
        ),
        scale: "small",
        category: "web3",
        tags: ["Extension", "Ethers.js", "TypeScript"],
        repo: "https://github.com/daniyar-sol/gas-tracker-mini",
        likes: 64,
        owner: "daniyar",
      },
      {
        slug: "tengebot",
        title: "TengeBot",
        summary: L(
          "A Telegram bot with live currency rates, converters and price alerts for tenge.",
          "Telegram-бот с актуальными курсами валют, конвертером и оповещениями по тенге.",
          "Теңгеге арналған валюта бағамдары, конвертер және баға хабарландырулары бар Telegram-бот.",
        ),
        description: L(
          "Over 4,000 users convert currencies and subscribe to alerts. Written in Go with Redis caching and runs on a single small VPS.",
          "Более 4000 пользователей конвертируют валюты и подписываются на оповещения. Написан на Go с кешем в Redis и работает на одном небольшом VPS.",
          "4000-нан астам пайдаланушы валюта айырбастайды және хабарландыруға жазылады. Go-да Redis кэшімен жазылған, бір шағын VPS-те жұмыс істейді.",
        ),
        scale: "small",
        category: "devtools",
        tags: ["Telegram", "Golang", "Redis"],
        repo: "https://github.com/alikhan-go/tengebot",
        likes: 92,
        owner: "alikhan",
      },
      {
        slug: "pixelquest",
        title: "PixelQuest",
        summary: L(
          "A tiny browser puzzle-platformer with hand-drawn pixel art and 30 levels.",
          "Крошечный браузерный платформер-головоломка с нарисованным вручную пиксель-артом и 30 уровнями.",
          "Қолмен салынған пиксель-арты және 30 деңгейі бар шағын браузерлік басқатырғыш-платформер.",
        ),
        description: L(
          "Made as a solo entry for a game jam. Canvas-based engine, gamepad support and a soundtrack composed by a friend.",
          "Сделана как сольная работа для гейм-джема. Движок на Canvas, поддержка геймпада и саундтрек, написанный другом.",
          "Гейм-джемге жеке жұмыс ретінде жасалған. Canvas негізіндегі қозғалтқыш, геймпад қолдауы және досы жазған саундтрек.",
        ),
        scale: "small",
        category: "game",
        tags: ["Canvas", "Game jam", "Pixel art"],
        repo: "https://github.com/madina-ui/pixelquest",
        demo: "https://pixelquest.example.com",
        likes: 118,
        owner: "madina",
      },
      {
        slug: "focusflow",
        title: "FocusFlow",
        summary: L(
          "A beautiful Flutter pomodoro timer with ambient sounds and weekly insights.",
          "Красивый pomodoro-таймер на Flutter с фоновыми звуками и еженедельной статистикой.",
          "Фондық дыбыстары мен апталық статистикасы бар әдемі Flutter pomodoro таймері.",
        ),
        description: L(
          "Built during Mobile Sprint: offline-first, Material 3, home-screen widgets and a calm design that students actually like using.",
          "Сделано на Mobile Sprint: offline-first, Material 3, виджеты на главном экране и спокойный дизайн, которым действительно приятно пользоваться студентам.",
          "Mobile Sprint кезінде жасалған: offline-first, Material 3, басты экран виджеттері және студенттер қуана қолданатын тыныш дизайн.",
        ),
        scale: "small",
        category: "mobile",
        tags: ["Flutter", "Dart", "Material 3"],
        repo: "https://github.com/timur-mobile/focusflow",
        likes: 77,
        owner: "timur",
      },
      {
        slug: "design-tokens-cli",
        title: "Design Tokens CLI",
        summary: L(
          "Sync Figma design tokens to Tailwind, CSS variables and Flutter themes with one command.",
          "Синхронизируйте токены Figma с Tailwind, CSS-переменными и темами Flutter одной командой.",
          "Figma токендерін Tailwind, CSS айнымалылары және Flutter тақырыптарымен бір пәрменмен синхрондаңыз.",
        ),
        description: L(
          "A zero-config CLI used by several teams to keep design and code in sync. Supports dark mode, semantic tokens and automatic changelogs.",
          "CLI без конфигурации, который несколько команд используют для синхронизации дизайна и кода. Поддерживает тёмную тему, семантические токены и автоматический changelog.",
          "Дизайн мен кодты синхронды ұстау үшін бірнеше команда қолданатын конфигурациясыз CLI. Қараңғы режим, семантикалық токендер және автоматты changelog қолдайды.",
        ),
        scale: "small",
        category: "devtools",
        tags: ["CLI", "Figma", "Tailwind"],
        repo: "https://github.com/madina-ui/design-tokens-cli",
        likes: 83,
        owner: "madina",
      },
      {
        slug: "team-match",
        title: "Team Match",
        summary: L(
          "Find hackathon teammates by skills, timezone and goals — a swipe-style matcher.",
          "Находите тиммейтов для хакатонов по навыкам, часовому поясу и целям — подбор в стиле свайпов.",
          "Хакатонға командаластарды дағдысы, уақыт белдеуі және мақсаты бойынша табыңыз — свайп стиліндегі іріктеу.",
        ),
        description: L(
          "Helps solo participants form balanced teams before an event. Built with Next.js and PostgreSQL, with a simple compatibility score.",
          "Помогает одиночным участникам собрать сбалансированные команды до события. Сделан на Next.js и PostgreSQL, с простой оценкой совместимости.",
          "Жеке қатысушыларға іс-шараға дейін теңгерімді команда құруға көмектеседі. Next.js және PostgreSQL негізінде, қарапайым үйлесімділік бағасымен.",
        ),
        scale: "small",
        category: "web",
        tags: ["Next.js", "PostgreSQL", "Community"],
        repo: "https://github.com/aruzhan-dev/team-match",
        demo: "https://teammatch.example.com",
        likes: 105,
        owner: "aruzhan",
        hack: "web3-builders-summit",
      },
    ];
    await tx.insert(projects).values(
      projectSeeds.map((p) => ({
        slug: p.slug,
        title: N(p.title),
        summary: p.summary,
        description: p.description,
        scale: p.scale,
        category: p.category,
        tags: p.tags,
        repoUrl: p.repo ?? null,
        demoUrl: p.demo ?? null,
        likes: p.likes,
        ownerId: dev[p.owner],
        hackathonId: p.hack ? hack[p.hack] : null,
        verified: Boolean(p.verified),
        txHash: p.verified ? fakeTxHash() : null,
        createdAt: new Date(Date.now() - (p.likes % 40) * DAY),
      })),
    );

    // ---------- Achievements ----------
    type A = {
      dev: string;
      hack: string | null;
      kind: string;
      place?: number;
      title: I18nText;
      issuer: string;
      daysAgo: number;
    };
    const ach: A[] = [
      { dev: "daniyar", hack: "web3-builders-summit", kind: "win", place: 1, title: L("Winner — Web3 Builders Summit (ChainFolio)", "Победитель — Web3 Builders Summit (ChainFolio)", "Жеңімпаз — Web3 Builders Summit (ChainFolio)"), issuer: "Web3 Kazakhstan", daysAgo: 108 },
      { dev: "daniyar", hack: "cybershield-ctf", kind: "finalist", title: L("Finalist — Smart contract track", "Финалист — трек смарт-контрактов", "Финалист — смарт-контракт тректі"), issuer: "CyberShield KZ", daysAgo: 44 },
      { dev: "daniyar", hack: null, kind: "certificate", title: L("Certified Solidity Developer", "Сертифицированный Solidity-разработчик", "Сертификатталған Solidity әзірлеушісі"), issuer: "HackChain Academy", daysAgo: 150 },
      { dev: "aruzhan", hack: "web3-builders-summit", kind: "win", place: 3, title: L("3rd place — Team Match", "3 место — Team Match", "3 орын — Team Match"), issuer: "Web3 Kazakhstan", daysAgo: 108 },
      { dev: "aruzhan", hack: "almaty-data-jam", kind: "participation", title: L("Participant — Almaty Data Jam", "Участник — Almaty Data Jam", "Қатысушы — Almaty Data Jam"), issuer: "Almaty Data Community", daysAgo: 199 },
      { dev: "aruzhan", hack: null, kind: "project", title: L("Verified open-source maintainer", "Подтверждённый open-source мейнтейнер", "Расталған open-source мейнтейнер"), issuer: "HackChain Foundation", daysAgo: 60 },
      { dev: "aigerim", hack: "almaty-data-jam", kind: "win", place: 1, title: L("Winner — Almaty Data Jam (SteppeGPT)", "Победитель — Almaty Data Jam (SteppeGPT)", "Жеңімпаз — Almaty Data Jam (SteppeGPT)"), issuer: "Almaty Data Community", daysAgo: 199 },
      { dev: "aigerim", hack: "web3-builders-summit", kind: "finalist", title: L("Finalist — AI x Web3 track", "Финалист — трек AI x Web3", "Финалист — AI x Web3 треегі"), issuer: "Web3 Kazakhstan", daysAgo: 109 },
      { dev: "nurzhan", hack: "cybershield-ctf", kind: "win", place: 1, title: L("Winner — CyberShield CTF Arena (SafeVault)", "Победитель — CyberShield CTF Arena (SafeVault)", "Жеңімпаз — CyberShield CTF Arena (SafeVault)"), issuer: "CyberShield KZ", daysAgo: 44 },
      { dev: "nurzhan", hack: null, kind: "certificate", title: L("Security Auditor badge", "Знак аудитора безопасности", "Қауіпсіздік аудиторы белгісі"), issuer: "HackChain Academy", daysAgo: 90 },
      { dev: "alikhan", hack: "web3-builders-summit", kind: "win", place: 2, title: L("2nd place — QazPay Gateway", "2 место — QazPay Gateway", "2 орын — QazPay Gateway"), issuer: "Web3 Kazakhstan", daysAgo: 108 },
      { dev: "alikhan", hack: "cybershield-ctf", kind: "participation", title: L("Participant — CyberShield CTF Arena", "Участник — CyberShield CTF Arena", "Қатысушы — CyberShield CTF Arena"), issuer: "CyberShield KZ", daysAgo: 45 },
      { dev: "madina", hack: "web3-builders-summit", kind: "win", title: L("Best UI/UX Award", "Награда за лучший UI/UX", "Үздік UI/UX марапаты"), issuer: "Web3 Kazakhstan", daysAgo: 108 },
      { dev: "madina", hack: "almaty-data-jam", kind: "participation", title: L("Participant — Almaty Data Jam", "Участник — Almaty Data Jam", "Қатысушы — Almaty Data Jam"), issuer: "Almaty Data Community", daysAgo: 199 },
      { dev: "timur", hack: "almaty-data-jam", kind: "finalist", title: L("Finalist — Almaty Data Jam", "Финалист — Almaty Data Jam", "Финалист — Almaty Data Jam"), issuer: "Almaty Data Community", daysAgo: 199 },
      { dev: "sofia", hack: "almaty-data-jam", kind: "win", place: 2, title: L("2nd place — AgroSense", "2 место — AgroSense", "2 орын — AgroSense"), issuer: "Almaty Data Community", daysAgo: 199 },
      { dev: "sofia", hack: null, kind: "project", title: L("AgroSense deployed on 12 farms", "AgroSense внедрён на 12 фермах", "AgroSense 12 шаруашылықта енгізілді"), issuer: "HackChain Foundation", daysAgo: 30 },
    ];
    await tx.insert(achievements).values(
      ach.map((a) => ({
        developerId: dev[a.dev],
        hackathonId: a.hack ? hack[a.hack] : null,
        kind: a.kind,
        title: a.title,
        issuer: a.issuer,
        place: a.place ?? null,
        txHash: fakeTxHash(),
        verified: true,
        awardedAt: new Date(Date.now() - a.daysAgo * DAY),
      })),
    );

    // ---------- Registrations ----------
    const pool = [
      "Dana Kaliyeva", "Ruslan Tulegenov", "Amina Zhunusova", "Yerlan Mukhamedov", "Kamila Aitbayeva",
      "Arman Dosov", "Zarina Nurgaliyeva", "Bakhyt Imanov", "Leila Ospanova", "Maxim Ivanov",
      "Dinara Sagyndykova", "Asset Kuanyshev", "Alina Kim", "Nursultan Bayzhanov", "Gulnara Abdrakhmanova",
      "Sanzhar Rakhimov", "Elena Smirnova", "Marat Kenzhebekov", "Aliya Temirkhanova", "Oleg Sidorov",
    ];
    const teams = ["Block Nomads", "Steppe Wolves", "Null Pointers", "Silk Road Labs", "Git Happens", "", "", ""];
    const levels = ["beginner", "middle", "middle", "pro"];
    const counts: Record<string, number> = {
      "hackchain-genesis": 14,
      "steppe-ai-for-good": 18,
      "astana-code-cup": 9,
      "mobile-sprint-48": 6,
      "cybershield-ctf": 16,
      "web3-builders-summit": 20,
      "almaty-data-jam": 12,
    };
    const regRows: (typeof registrations.$inferInsert)[] = [];
    for (const [slug, c] of Object.entries(counts)) {
      pool.slice(0, c).forEach((name, i) => {
        regRows.push({
          hackathonId: hack[slug],
          name,
          email: `${name.toLowerCase().replace(/[^a-z]+/g, ".")}@example.com`,
          teamName: teams[(i + slug.length) % teams.length],
          experience: levels[(i + slug.length) % levels.length],
        });
      });
    }
    // developers registered to events they have credentials for + upcoming ones
    const devEmail = (h: string) => `${h}@hackchain.dev`;
    const devNames = Object.fromEntries(devRows.map((d) => [d.handle, d.name]));
    const addDevReg = (h: string, slug: string) => {
      if (regRows.some((r) => r.hackathonId === hack[slug] && r.email === devEmail(h))) return;
      regRows.push({
        hackathonId: hack[slug],
        developerId: dev[h],
        name: devNames[h],
        email: devEmail(h),
        teamName: "",
        experience: "pro",
      });
    };
    for (const a of ach) if (a.hack) addDevReg(a.dev, a.hack);
    for (const h of Object.keys(dev)) {
      addDevReg(h, "hackchain-genesis");
      if (["aigerim", "sofia", "aruzhan", "timur"].includes(h)) addDevReg(h, "steppe-ai-for-good");
    }
    await tx.insert(registrations).values(regRows).onConflictDoNothing();
  });
}
import { translate } from "./dict";
import { getDevelopers, getHackathons, getProjects } from "./data";
import {
  pickText,
  type DeveloperDTO,
  type HackathonDTO,
  type Locale,
  type ProjectDTO,
} from "./types";

export interface AssistantLink {
  label: string;
  href: string;
}
export interface AssistantReply {
  reply: string;
  links: AssistantLink[];
  action?: { type: "navigate"; href: string };
  suggestions?: string[];
  source: "builtin" | "llm";
}
export interface HistoryItem {
  role: "user" | "assistant";
  text: string;
}

/* ------------------------------------------------------------------ */
/* Text helpers                                                        */
/* ------------------------------------------------------------------ */

function norm(s: string): string {
  return s
    .toLowerCase()
    .replace(/ё/g, "е")
    .replace(/[^\p{L}\p{N}\s@+.#-]/gu, " ")
    .replace(/\s+/g, " ")
    .trim();
}

const ALIAS: Record<string, string> = {
  алматы: "almaty",
  алмата: "almaty",
  астана: "astana",
  астане: "astana",
  шымкент: "shymkent",
  караганда: "karaganda",
  қарағанды: "karaganda",
  бишкек: "bishkek",
  питон: "python",
  пайтон: "python",
  солидити: "solidity",
  реакт: "react",
  флаттер: "flutter",
  раст: "rust",
  докер: "docker",
  безопасность: "security",
  безопасности: "security",
  қауіпсіздік: "security",
  кибербезопасность: "cybersecurity",
  нейросети: "ai",
  нейросеть: "ai",
  ии: "ai",
  жи: "ai",
  веб3: "web3",
};

function tokenize(t: string): string[] {
  return t
    .split(" ")
    .filter(Boolean)
    .map((w) => ALIAS[w] ?? w);
}

const has = (t: string, stems: string[]) => stems.some((s) => t.includes(s));

const W = {
  hack: ["хакатон", "hackathon", "турнир", "tournament", "жарыс", "соревнован", "competition", "contest", "ctf", "байқау", "чемпионат", "event", "мероприят", "іс-шара"],
  proj: ["проект", "project", "жоба", "приложен", "стартап", "startup", "продукт", "product", "өнім"],
  dev: ["разработчик", "developer", "программист", "programmer", "инженер", "engineer", "әзірлеуш", "бағдарламашы", "талант", "специалист", "specialist", "маман", "people", "люди", "адамдар", "dev "],
  port: ["портфолио", "portfolio", "профил", "profile", "резюме", "resume", " cv ", " био", " bio "],
  create: ["созда", "организ", "провест", "устро", "запуст", "сделать", "host", "create", "organi", "launch", "start a", "make a", "setup", "set up", "құр", "ұйымдастыр", "өткіз", "жаса"],
  add: ["добав", "опубликов", "загруз", "разместит", "submit", "add ", "upload", "publish", "post ", "қос", "жари", "жібер", "орналастыр"],
  reg: ["регистр", "участв", "записат", "register", "sign up", "signup", "join", "enroll", "apply", "тіркел", "қатыс", "жазыл"],
  nav: [" открой", " открыть", " перейд", " зайди", " open", " go to", " navigate", " take me", " ашы", " аш ", " өт ", " өтіп", " ашып"],
  verify: ["блокчейн", "blockchain", "верифи", "подтвержд", "verify", "verif", "on-chain", "onchain", "сертификат", "certificate", "тексер", "растау", "расталған", "хэш", "hash", "транзакц", "transaction", "подлинн", "проверк", "проверить"],
  about: ["что такое", "что это", "что за", "о платформе", "расскажи о", "расскажи про", "хакчейн", "hackchain", "what is", "what's", "about", "who are you", "кто ты", "кто вы", "не туралы", "деген не", "қандай платформа", "кімсің", "how does it work", "how it works", "как работает", "қалай жұмыс"],
  top: ["лучш", "топ", "top", "best", "рейтинг", "ranking", "leaderboard", "репутац", "reputation", "үздік", "бедел", "сильнейш", "strongest"],
  up: ["ближайш", "скоро", "предстоящ", "upcoming", "next", "soon", "жақын", "алдағы", "будущ"],
  live: ["сейчас", "идет", "идёт", "live", " now", "ongoing", "current", "қазір", "жүріп"],
  past: ["прошедш", "прошл", "past", "previous", "finished", "аяқталған", "өткен"],
  large: ["масштаб", "больш", "крупн", "large", "big", "enterprise", "ірі", "ауқымды", "үлкен"],
  small: ["малень", "небольш", "мелк", "small", "mini", "tiny", "шағын", "кішкентай", "кіші"],
  thanks: ["спасибо", "благодар", "thank", "рахмет"],
  bye: [" пока ", "до свидан", " bye", "goodbye", "see you", "сау бол", "көріскенше"],
  lang: ["язык", "language", " тіл", "перевод", "translate"],
  help: ["помощь", "помоги", "что умеешь", "что ты умеешь", "help", "what can you", "can you do", "көмек", "не істей аласың", "не білесің"],
  home: ["главн", "home", "басты"],
};

const GREET_WORDS = ["hi", "hey", "hello", "привет", "здравствуйте", "здравствуй", "хай", "салем", "сәлем", "сәлеметсіз", "сәлеметсізбе", "салам", "добрый", "доброе"];

/* ------------------------------------------------------------------ */
/* Localized texts                                                     */
/* ------------------------------------------------------------------ */

interface Texts {
  greet: string;
  thanks: string;
  bye: string;
  help: string;
  about: string;
  verify: string;
  createHack: string;
  createProfile: string;
  submitProject: string;
  register: string;
  lang: string;
  fallback: string;
  noHack: string;
  noProj: string;
  noDev: string;
  found: string;
  opening: (label: string) => string;
  hackIntro: (kind: "upcoming" | "live" | "past" | "active") => string;
  projIntro: (scale: "large" | "small" | "all") => string;
  devIntro: string;
  devDetail: (d: DeveloperDTO, role: string, rep: number) => string;
  projDetail: (title: string, summary: string, owner: string | null, likes: number) => string;
  hackDetail: (title: string, when: string, where: string, prize: string, reg: number, max: number, status: HackathonDTO["status"]) => string;
}

const TXT: Record<Locale, Texts> = {
  en: {
    greet: "Hi! I'm the HackChain assistant. Ask me about tournaments, projects or developers — or say “open hackathons”.",
    thanks: "You're welcome! Anything else I can help with?",
    bye: "See you soon on HackChain!",
    help: "I can list upcoming hackathons, find projects (small or large), search developers by skill or city, explain how to host a tournament, register or create a portfolio, and open pages for you. Try “Python developers” or “open projects”.",
    about: "HackChain is a Web3 platform for verifiable developer reputation. Hackathon results, wins and projects are confirmed by organizers and recorded on the blockchain, so your portfolio is a proof that employers and communities can trust. You can join hackathons, host your own IT tournament, publish projects and share your portfolio.",
    verify: "When an organizer confirms an achievement, HackChain records it with a transaction hash. Paste that hash on the Verify page to check it's real. Every portfolio shows its verified credentials and a reputation score.",
    createHack: "To host your own tournament: open the form, enter the name, description, dates, format and prize pool, then launch it. People can register immediately and compete.",
    createProfile: "Creating a portfolio takes a minute: add your name, role, short bio and skills. Organizers can then verify your achievements and add them to your profile.",
    submitProject: "To publish a project: choose your portfolio as the author, describe the project, pick small or large scale and add links. It appears in the catalog instantly.",
    register: "Here are the events you can join right now:",
    lang: "Use the language switcher in the top bar. HackChain works in English, Русский and Қазақша — and I understand all three.",
    fallback: "I'm not sure I got that. I can search tournaments, projects and developers, or explain how HackChain works. Try “upcoming hackathons” or “Solidity developers”.",
    noHack: "I couldn't find matching events right now. You can host your own tournament!",
    noProj: "I couldn't find matching projects yet. You can publish yours!",
    noDev: "I couldn't find matching developers.",
    found: "Here's what I found:",
    opening: (l) => `Opening ${l}…`,
    hackIntro: (k) =>
      k === "past" ? "Here are recent finished events:" : k === "live" ? "These events are live right now:" : k === "upcoming" ? "Here are the upcoming events:" : "Here are the events to join:",
    projIntro: (s) => (s === "large" ? "Here are large-scale projects:" : s === "small" ? "Here are small projects:" : "Here are popular projects:"),
    devIntro: "Here are the top developers by verified reputation:",
    devDetail: (d, role, rep) => `${d.name} — ${role}${d.location ? ", " + d.location : ""}. Reputation: ${rep} points, ${d.achievementsCount} verified achievements, ${d.projectsCount} projects. Skills: ${d.skills.slice(0, 5).join(", ")}.`,
    projDetail: (t, s, o, l) => `${t} — ${s}${o ? " Author: " + o + "." : ""} ${l} likes.`,
    hackDetail: (t, w, wh, p, r, m, st) =>
      `${t} — ${w}${wh ? ", " + wh : ""}.${p ? " Prize pool: " + p + "." : ""} ${r} of ${m} spots taken.${st === "past" ? " This event has finished." : " You can register on the event page."}`,
  },
  ru: {
    greet: "Привет! Я ассистент HackChain. Спросите меня о турнирах, проектах или разработчиках — или скажите «открой хакатоны».",
    thanks: "Пожалуйста! Чем ещё могу помочь?",
    bye: "До скорой встречи на HackChain!",
    help: "Я могу показать ближайшие хакатоны, найти проекты (небольшие или масштабные), подобрать разработчиков по навыку или городу, объяснить, как создать турнир, зарегистрироваться или сделать портфолио, и открыть нужные страницы. Попробуйте «Python разработчики» или «открой проекты».",
    about: "HackChain — это Web3-платформа проверяемой репутации разработчиков. Результаты хакатонов, победы и проекты подтверждаются организаторами и записываются в блокчейн, поэтому ваше портфолио — это доказательство, которому доверяют работодатели и сообщества. Здесь можно участвовать в хакатонах, создавать свои IT-турниры, публиковать проекты и делиться портфолио.",
    verify: "Когда организатор подтверждает достижение, HackChain записывает его с хэшем транзакции. Вставьте этот хэш на странице проверки, чтобы убедиться, что награда настоящая. В каждом портфолио видны подтверждённые награды и рейтинг репутации.",
    createHack: "Чтобы создать свой турнир, откройте форму, укажите название, описание, даты, формат и призовой фонд — и запускайте. Люди смогут сразу регистрироваться и соревноваться.",
    createProfile: "Портфолио создаётся за минуту: укажите имя, специализацию, короткое био и навыки. Затем организаторы смогут подтвердить ваши достижения и добавить их в профиль.",
    submitProject: "Чтобы опубликовать проект: выберите своё портфолио как автора, опишите проект, укажите масштаб — небольшой или масштабный — и добавьте ссылки. Он сразу появится в каталоге.",
    register: "Вот события, к которым можно присоединиться прямо сейчас:",
    lang: "Переключайте язык в верхней панели. HackChain работает на английском, русском и казахском — и я понимаю все три.",
    fallback: "Кажется, я не совсем понял. Я могу искать турниры, проекты и разработчиков или объяснить, как работает HackChain. Попробуйте «ближайшие хакатоны» или «Solidity разработчики».",
    noHack: "Сейчас подходящих событий не нашёл. Можно создать свой турнир!",
    noProj: "Подходящих проектов пока нет. Опубликуйте свой!",
    noDev: "Подходящих разработчиков не нашёл.",
    found: "Вот что я нашёл:",
    opening: (l) => `Открываю: ${l}…`,
    hackIntro: (k) =>
      k === "past" ? "Вот недавно завершённые события:" : k === "live" ? "Эти события идут прямо сейчас:" : k === "upcoming" ? "Вот ближайшие события:" : "Вот события, к которым можно присоединиться:",
    projIntro: (s) => (s === "large" ? "Вот масштабные проекты:" : s === "small" ? "Вот небольшие проекты:" : "Вот популярные проекты:"),
    devIntro: "Вот лучшие разработчики по подтверждённой репутации:",
    devDetail: (d, role, rep) => `${d.name} — ${role}${d.location ? ", " + d.location : ""}. Репутация: ${rep} очков, подтверждённых достижений: ${d.achievementsCount}, проектов: ${d.projectsCount}. Навыки: ${d.skills.slice(0, 5).join(", ")}.`,
    projDetail: (t, s, o, l) => `${t} — ${s}${o ? " Автор: " + o + "." : ""} Лайков: ${l}.`,
    hackDetail: (t, w, wh, p, r, m, st) =>
      `${t} — ${w}${wh ? ", " + wh : ""}.${p ? " Призовой фонд: " + p + "." : ""} Занято мест: ${r} из ${m}.${st === "past" ? " Событие завершено." : " Зарегистрироваться можно на странице события."}`,
  },
  kk: {
    greet: "Сәлем! Мен HackChain көмекшісімін. Турнирлер, жобалар немесе әзірлеушілер туралы сұраңыз — немесе «хакатондарды аш» деп айтыңыз.",
    thanks: "Оқасы жоқ! Тағы немен көмектесе аламын?",
    bye: "HackChain-де жақында кездескенше!",
    help: "Мен жақын хакатондарды көрсете аламын, жобаларды (шағын немесе ауқымды) табамын, әзірлеушілерді дағдысы немесе қаласы бойынша іздеймін, турнир ұйымдастыру, тіркелу және портфолио жасау жолын түсіндіремін, керекті беттерді ашамын. «Python әзірлеушілер» немесе «жобаларды аш» деп көріңіз.",
    about: "HackChain — әзірлеушілердің тексерілетін беделіне арналған Web3-платформа. Хакатон нәтижелері, жеңістер мен жобалар ұйымдастырушылармен расталып, блокчейнге жазылады, сондықтан портфолиоңыз — жұмыс берушілер мен қауымдастықтар сенетін дәлел. Мұнда хакатондарға қатысуға, өз IT-турниріңізді ұйымдастыруға, жобаларды жариялауға және портфолионы бөлісуге болады.",
    verify: "Ұйымдастырушы жетістікті растағанда, HackChain оны транзакция хэшімен жазады. Марапаттың шынайы екеніне көз жеткізу үшін осы хэшті тексеру бетіне қойыңыз. Әр портфолиода расталған марапаттар мен бедел рейтингі көрсетіледі.",
    createHack: "Өз турниріңізді ұйымдастыру үшін форманы ашып, атауын, сипаттамасын, күндерін, форматын және жүлде қорын енгізіп, іске қосыңыз. Адамдар бірден тіркеліп, жарыса алады.",
    createProfile: "Портфолио бір минутта жасалады: атыңызды, мамандығыңызды, қысқаша био мен дағдыларыңызды енгізіңіз. Содан кейін ұйымдастырушылар жетістіктеріңізді растап, профильге қоса алады.",
    submitProject: "Жобаны жариялау үшін: автор ретінде өз портфолиоңызды таңдаңыз, жобаны сипаттаңыз, ауқымын — шағын немесе ауқымды — таңдап, сілтемелерді қосыңыз. Ол каталогта бірден пайда болады.",
    register: "Мына іс-шараларға дәл қазір қосыла аласыз:",
    lang: "Тілді жоғарғы панельден ауыстырыңыз. HackChain ағылшын, орыс және қазақ тілдерінде жұмыс істейді — мен үшеуін де түсінемін.",
    fallback: "Түсіне алмадым білем. Мен турнирлерді, жобаларды және әзірлеушілерді іздей аламын немесе HackChain қалай жұмыс істейтінін түсіндіре аламын. «Жақын хакатондар» немесе «Solidity әзірлеушілер» деп көріңіз.",
    noHack: "Қазір сәйкес іс-шара таппадым. Өз турниріңізді ұйымдастыра аласыз!",
    noProj: "Сәйкес жобалар әзірге жоқ. Өзіңіздікін жариялаңыз!",
    noDev: "Сәйкес әзірлеушілерді таппадым.",
    found: "Мынаны таптым:",
    opening: (l) => `Ашып жатырмын: ${l}…`,
    hackIntro: (k) =>
      k === "past" ? "Жақында аяқталған іс-шаралар:" : k === "live" ? "Қазір өтіп жатқан іс-шаралар:" : k === "upcoming" ? "Жақын іс-шаралар:" : "Қосылуға болатын іс-шаралар:",
    projIntro: (s) => (s === "large" ? "Ауқымды жобалар:" : s === "small" ? "Шағын жобалар:" : "Танымал жобалар:"),
    devIntro: "Расталған бедел бойынша үздік әзірлеушілер:",
    devDetail: (d, role, rep) => `${d.name} — ${role}${d.location ? ", " + d.location : ""}. Бедел: ${rep} ұпай, расталған жетістік: ${d.achievementsCount}, жоба: ${d.projectsCount}. Дағдылары: ${d.skills.slice(0, 5).join(", ")}.`,
    projDetail: (t, s, o, l) => `${t} — ${s}${o ? " Автор: " + o + "." : ""} Ұнатулар: ${l}.`,
    hackDetail: (t, w, wh, p, r, m, st) =>
      `${t} — ${w}${wh ? ", " + wh : ""}.${p ? " Жүлде қоры: " + p + "." : ""} Орын толды: ${r} / ${m}.${st === "past" ? " Іс-шара аяқталды." : " Іс-шара бетінде тіркелуге болады."}`,
  },
};

const LANG_TAG: Record<Locale, string> = { en: "en-US", ru: "ru-RU", kk: "kk-KZ" };

function fmtDate(iso: string, locale: Locale): string {
  try {
    return new Date(iso).toLocaleDateString(LANG_TAG[locale], { day: "numeric", month: "short", timeZone: "UTC" });
  } catch {
    return iso.slice(0, 10);
  }
}

function whenText(h: HackathonDTO, locale: Locale): string {
  const a = fmtDate(h.startsAt, locale);
  const b = fmtDate(h.endsAt, locale);
  return a === b ? a : `${a} – ${b}`;
}

function formatLabel(f: HackathonDTO["format"], locale: Locale) {
  return translate(locale, f === "online" ? "common.online" : f === "offline" ? "common.offline" : "common.hybrid");
}

/* ------------------------------------------------------------------ */
/* Scoring                                                             */
/* ------------------------------------------------------------------ */

function skillHit(k: string, T: string, toks: string[]) {
  return k.length >= 3 && (toks.includes(k) || T.includes(` ${k} `));
}

function scoreDev(d: DeveloperDTO, T: string, toks: string[]): number {
  let s = 0;
  const name = norm(d.name);
  if (T.includes(` ${name} `)) s += 12;
  else for (const part of name.split(" ")) if (part.length >= 3 && toks.includes(part)) s += 10;
  if (toks.includes(d.handle)) s += 10;
  for (const sk of d.skills) if (skillHit(norm(sk), T, toks)) s += 4;
  for (const part of norm(d.location).split(" ")) if (part.length >= 4 && toks.includes(part)) s += 3;
  return s;
}

function scoreProject(p: ProjectDTO, T: string, toks: string[]): number {
  let s = 0;
  const title = norm(p.title.en);
  if (T.includes(` ${title} `)) s += 12;
  for (const tag of p.tags) if (skillHit(norm(tag), T, toks)) s += 4;
  if (toks.includes(p.category)) s += 3;
  if (p.ownerName) for (const part of norm(p.ownerName).split(" ")) if (part.length >= 3 && toks.includes(part)) s += 4;
  return s;
}

const HACK_STOP = new Set(["hackathon", "tournament", "arena", "cup", "2026"]);
function scoreHack(h: HackathonDTO, T: string, toks: string[]): number {
  let s = 0;
  const title = norm(h.title.en);
  if (T.includes(` ${title} `)) s += 12;
  else
    for (const part of title.split(" ")) {
      if (part.length >= 4 && !HACK_STOP.has(part) && toks.includes(part)) s += 5;
    }
  for (const tag of h.tags) if (skillHit(norm(tag), T, toks)) s += 3;
  if (h.location && norm(h.location).split(" ").some((p) => p.length >= 4 && toks.includes(p))) s += 2;
  return s;
}

/* ------------------------------------------------------------------ */
/* Main entry                                                          */
/* ------------------------------------------------------------------ */

function lines(items: string[]) {
  return items.map((i) => `• ${i}`).join("\n");
}

export async function answer(
  message: string,
  locale: Locale,
  history: HistoryItem[] = [],
): Promise<AssistantReply> {
  const msg = message.trim().slice(0, 500);
  const tx = TXT[locale];
  const t = (k: Parameters<typeof translate>[1]) => translate(locale, k);
  const sugg = [t("ai.s1"), t("ai.s2"), t("ai.s3"), t("ai.s4")];
  const reply = (r: Partial<AssistantReply> & { reply: string }): AssistantReply => ({
    links: [],
    source: "builtin",
    ...r,
  });

  const base = norm(msg);
  if (!base) return reply({ reply: tx.help, suggestions: sugg });
  const T = ` ${base} `;
  const toks = tokenize(base);

  const wantHack = has(T, W.hack);
  const wantProj = has(T, W.proj);
  const wantDev = has(T, W.dev) || has(T, W.port);
  const wantCreate = has(T, W.create);
  const wantAdd = has(T, W.add);
  const wantNav = has(T, W.nav);

  // greetings / small talk (short messages only)
  const shortMsg = toks.length <= 5;
  if (shortMsg && toks.some((w) => GREET_WORDS.includes(w)) && !wantHack && !wantProj && !wantDev) {
    return reply({ reply: tx.greet, suggestions: sugg });
  }
  if (shortMsg && has(T, W.thanks)) return reply({ reply: tx.thanks });
  if (shortMsg && has(T, W.bye) && !has(T, W.past)) return reply({ reply: tx.bye });

  // navigation commands
  const navTargetFor = (): { href: string; label: string } | null => {
    if (has(T, W.verify) && !wantCreate) return { href: "/verify", label: t("nav.verify") };
    if (wantHack && wantCreate) return { href: "/hackathons/new", label: t("nav.hostTournament") };
    if (has(T, W.port) && (wantCreate || wantAdd)) return { href: "/developers/new", label: t("nav.createProfile") };
    if (wantProj && (wantAdd || wantCreate)) return { href: "/projects/new", label: t("form.projectTitle") };
    if (wantHack) return { href: "/hackathons", label: t("nav.hackathons") };
    if (wantProj) return { href: "/projects", label: t("nav.projects") };
    if (wantDev) return { href: "/developers", label: t("nav.developers") };
    if (has(T, W.home)) return { href: "/", label: t("nav.home") };
    return null;
  };
  if (wantNav) {
    const target = navTargetFor();
    if (target) {
      return reply({
        reply: tx.opening(target.label),
        links: [{ label: target.label, href: target.href }],
        action: { type: "navigate", href: target.href },
      });
    }
  }

  const [devs, projs, hacks] = await Promise.all([getDevelopers(), getProjects(), getHackathons()]);

  // best entity matches
  const dScores = devs.map((d) => ({ d, s: scoreDev(d, T, toks) })).filter((x) => x.s > 0).sort((a, b) => b.s - a.s);
  const pScores = projs.map((p) => ({ p, s: scoreProject(p, T, toks) })).filter((x) => x.s > 0).sort((a, b) => b.s - a.s);
  const hScores = hacks.map((h) => ({ h, s: scoreHack(h, T, toks) })).filter((x) => x.s > 0).sort((a, b) => b.s - a.s);

  const topD = dScores[0];
  const topP = pScores[0];
  const topH = hScores[0];
  const bestExact = Math.max(topD && topD.s >= 10 ? topD.s : 0, topP && topP.s >= 10 ? topP.s : 0, topH && (topH.s >= 10 || (topH.s >= 5 && wantHack)) ? topH.s : 0);

  const wantsCreateFlow = (wantHack && wantCreate) || (has(T, W.port) && (wantCreate || wantAdd)) || (wantProj && (wantAdd || wantCreate));

  if (bestExact > 0 && !wantsCreateFlow) {
    if (topD && topD.s === bestExact) {
      const d = topD.d;
      return reply({
        reply: tx.devDetail(d, pickText(d.role, locale), d.reputation),
        links: [{ label: `${t("common.viewPortfolio")}: ${d.name}`, href: `/developers/${d.handle}` }],
      });
    }
    if (topP && topP.s === bestExact) {
      const p = topP.p;
      return reply({
        reply: tx.projDetail(pickText(p.title, locale), pickText(p.summary, locale), p.ownerName, p.likes),
        links: [{ label: pickText(p.title, locale), href: `/projects/${p.slug}` }],
      });
    }
    if (topH) {
      const h = topH.h;
      return reply({
        reply: tx.hackDetail(
          pickText(h.title, locale),
          whenText(h, locale),
          [formatLabel(h.format, locale), h.location].filter(Boolean).join(", "),
          h.prizePool,
          h.participants,
          h.maxParticipants,
          h.status,
        ),
        links: [{ label: pickText(h.title, locale), href: `/hackathons/${h.slug}` }],
      });
    }
  }

  // creation flows
  if (wantHack && wantCreate) {
    return reply({ reply: tx.createHack, links: [{ label: t("nav.hostTournament"), href: "/hackathons/new" }] });
  }
  if (has(T, W.port) && (wantCreate || wantAdd)) {
    return reply({ reply: tx.createProfile, links: [{ label: t("nav.createProfile"), href: "/developers/new" }] });
  }
  if (wantProj && (wantAdd || wantCreate)) {
    return reply({ reply: tx.submitProject, links: [{ label: t("form.projectTitle"), href: "/projects/new" }] });
  }

  const hackLink = (h: HackathonDTO): AssistantLink => ({ label: pickText(h.title, locale), href: `/hackathons/${h.slug}` });
  const hackLine = (h: HackathonDTO) =>
    `${pickText(h.title, locale)} — ${whenText(h, locale)}, ${formatLabel(h.format, locale)}${h.prizePool ? ", " + h.prizePool : ""}`;

  // registration
  if (has(T, W.reg)) {
    const open = hacks.filter((h) => h.status !== "past" && h.participants < h.maxParticipants).slice(0, 4);
    if (open.length) {
      return reply({ reply: `${tx.register}\n${lines(open.map(hackLine))}`, links: open.map(hackLink) });
    }
    return reply({ reply: tx.noHack, links: [{ label: t("nav.hostTournament"), href: "/hackathons/new" }] });
  }

  if (has(T, W.verify)) {
    return reply({ reply: tx.verify, links: [{ label: t("nav.verify"), href: "/verify" }] });
  }

  if (has(T, W.about) && !wantHack && !wantProj && !wantDev) {
    return reply({ reply: tx.about, links: [{ label: t("nav.hackathons"), href: "/hackathons" }, { label: t("nav.projects"), href: "/projects" }] });
  }
  if (has(T, W.about) && /hackchain|хакчейн/.test(base) && toks.length <= 5) {
    return reply({ reply: tx.about, links: [{ label: t("nav.hackathons"), href: "/hackathons" }] });
  }

  if (has(T, W.lang)) return reply({ reply: tx.lang });
  if (has(T, W.help)) return reply({ reply: tx.help, suggestions: sugg });

  // partial entity matches (skills, tags, cities)
  const any = wantHack || wantProj || wantDev;
  const groupsD = dScores.filter((x) => x.s >= 3).slice(0, 4);
  const groupsP = pScores.filter((x) => x.s >= 3).slice(0, 4);
  const groupsH = hScores.filter((x) => x.s >= 3).slice(0, 4);
  if ((groupsD.length && (wantDev || !any)) || (groupsP.length && (wantProj || !any)) || (groupsH.length && (wantHack || !any))) {
    const out: string[] = [];
    const links: AssistantLink[] = [];
    if (groupsD.length && (wantDev || !any)) {
      for (const { d } of groupsD) {
        out.push(`${d.name} — ${pickText(d.role, locale)}`);
        links.push({ label: d.name, href: `/developers/${d.handle}` });
      }
    }
    if (groupsP.length && (wantProj || !any)) {
      for (const { p } of groupsP.slice(0, 3)) {
        out.push(`${pickText(p.title, locale)} — ${pickText(p.summary, locale)}`);
        links.push({ label: pickText(p.title, locale), href: `/projects/${p.slug}` });
      }
    }
    if (groupsH.length && (wantHack || !any)) {
      for (const { h } of groupsH.slice(0, 3)) {
        out.push(hackLine(h));
        links.push(hackLink(h));
      }
    }
    return reply({ reply: `${tx.found}\n${lines(out)}`, links: links.slice(0, 6) });
  }

  // generic lists
  if (wantDev && has(T, W.top)) {
    const top = devs.slice(0, 4);
    return reply({
      reply: `${tx.devIntro}\n${lines(top.map((d) => `${d.name} — ${pickText(d.role, locale)}, ${d.reputation} ${t("common.points")}`))}`,
      links: top.map((d) => ({ label: d.name, href: `/developers/${d.handle}` })),
    });
  }
  if (wantHack) {
    const kind = has(T, W.past) ? "past" : has(T, W.live) ? "live" : has(T, W.up) ? "upcoming" : "active";
    let list = hacks.filter((h) => (kind === "active" ? h.status !== "past" : h.status === kind));
    if (kind === "active") list = list.slice(0, 5);
    list = list.slice(0, 5);
    if (!list.length) return reply({ reply: tx.noHack, links: [{ label: t("nav.hostTournament"), href: "/hackathons/new" }] });
    return reply({ reply: `${tx.hackIntro(kind)}\n${lines(list.map(hackLine))}`, links: list.map(hackLink) });
  }
  if (wantProj) {
    const scale = has(T, W.large) ? "large" : has(T, W.small) ? "small" : "all";
    const list = projs.filter((p) => scale === "all" || p.scale === scale).slice(0, 4);
    if (!list.length) return reply({ reply: tx.noProj, links: [{ label: t("form.projectTitle"), href: "/projects/new" }] });
    return reply({
      reply: `${tx.projIntro(scale)}\n${lines(list.map((p) => `${pickText(p.title, locale)} — ${pickText(p.summary, locale)}`))}`,
      links: list.map((p) => ({ label: pickText(p.title, locale), href: `/projects/${p.slug}` })),
    });
  }
  if (wantDev || has(T, W.top)) {
    const top = devs.slice(0, 4);
    if (!top.length) return reply({ reply: tx.noDev });
    return reply({
      reply: `${tx.devIntro}\n${lines(top.map((d) => `${d.name} — ${pickText(d.role, locale)}, ${d.reputation} ${t("common.points")}`))}`,
      links: top.map((d) => ({ label: d.name, href: `/developers/${d.handle}` })),
    });
  }

  if (has(T, W.about)) return reply({ reply: tx.about });

  // free-form question: use an LLM if one is configured
  const llm = await askLLM(msg, locale, history, { devs, projs, hacks });
  if (llm) return reply({ reply: llm, source: "llm" });

  return reply({ reply: tx.fallback, suggestions: sugg });
}

/* ------------------------------------------------------------------ */
/* Optional LLM fallback (OpenAI-compatible)                           */
/* ------------------------------------------------------------------ */

async function askLLM(
  message: string,
  locale: Locale,
  history: HistoryItem[],
  data: { devs: DeveloperDTO[]; projs: ProjectDTO[]; hacks: HackathonDTO[] },
): Promise<string | null> {
  const key = process.env.OPENAI_API_KEY;
  if (!key) return null;
  const baseUrl = (process.env.OPENAI_BASE_URL ?? "https://api.openai.com/v1").replace(/\/$/, "");
  const model = process.env.OPENAI_MODEL ?? "gpt-4o-mini";
  const langName = locale === "ru" ? "Russian" : locale === "kk" ? "Kazakh" : "English";

  const context = [
    "Hackathons: " + data.hacks.slice(0, 8).map((h) => `${h.title.en} (${h.status}, ${h.startsAt.slice(0, 10)}, ${h.format}, prize ${h.prizePool})`).join("; "),
    "Projects: " + data.projs.slice(0, 10).map((p) => `${p.title.en} [${p.scale}, ${p.category}]`).join("; "),
    "Developers: " + data.devs.slice(0, 8).map((d) => `${d.name} (${d.role.en}, ${d.reputation} pts)`).join("; "),
  ].join("\n");

  const system = `You are the HackChain assistant. HackChain is a Web3 platform for verifiable developer reputation: hackathon wins and projects are confirmed by organizers and recorded on-chain. Users can browse hackathons/tournaments, host their own, register, publish projects (small or large scale), and build portfolios. Reply in ${langName}, concisely (max 4 sentences), friendly, plain text. Use this live data when relevant:\n${context}`;

  try {
    const ctrl = new AbortController();
    const timer = setTimeout(() => ctrl.abort(), 12000);
    const res = await fetch(`${baseUrl}/chat/completions`, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${key}` },
      body: JSON.stringify({
        model,
        temperature: 0.5,
        max_tokens: 300,
        messages: [
          { role: "system", content: system },
          ...history.slice(-6).map((h) => ({ role: h.role, content: h.text.slice(0, 500) })),
          { role: "user", content: message },
        ],
      }),
      signal: ctrl.signal,
    });
    clearTimeout(timer);
    if (!res.ok) return null;
    const json = (await res.json()) as { choices?: { message?: { content?: string } }[] };
    return json.choices?.[0]?.message?.content?.trim() || null;
  } catch {
    return null;
  }
}
import { count, desc, eq, sql } from "drizzle-orm";
import { db } from "@/db";
import { ensureSeeded } from "@/db/seed";
import {
  achievements,
  developers,
  hackathons,
  projects,
  registrations,
} from "@/db/schema";
import {
  achievementPoints,
  computeStatus,
  type AchievementDTO,
  type AchievementKind,
  type DeveloperDTO,
  type HackathonDTO,
  type HackathonFormat,
  type ParticipantDTO,
  type ProjectDTO,
  type ProjectScale,
  type StatsDTO,
} from "./types";

type DevRow = typeof developers.$inferSelect;

function toDeveloper(
  d: DevRow,
  achs: { kind: string; place: number | null; verified: boolean }[],
  projectsCount: number,
): DeveloperDTO {
  const verified = achs.filter((a) => a.verified);
  return {
    id: d.id,
    handle: d.handle,
    name: d.name,
    role: d.role,
    bio: d.bio,
    location: d.location,
    skills: d.skills,
    github: d.github,
    website: d.website,
    telegram: d.telegram,
    wallet: d.wallet,
    openToWork: d.openToWork,
    reputation: verified.reduce((sum, a) => sum + achievementPoints(a.kind, a.place), 0),
    achievementsCount: verified.length,
    projectsCount,
    winsCount: verified.filter((a) => a.kind === "win").length,
    joinedAt: d.createdAt.toISOString(),
  };
}

export async function getDevelopers(): Promise<DeveloperDTO[]> {
  await ensureSeeded();
  const [devs, achs, pcs] = await Promise.all([
    db.select().from(developers),
    db
      .select({
        developerId: achievements.developerId,
        kind: achievements.kind,
        place: achievements.place,
        verified: achievements.verified,
      })
      .from(achievements),
    db
      .select({ ownerId: projects.ownerId, n: count() })
      .from(projects)
      .groupBy(projects.ownerId),
  ]);
  const pcMap = new Map<number, number>();
  for (const p of pcs) if (p.ownerId != null) pcMap.set(p.ownerId, Number(p.n));
  return devs
    .map((d) =>
      toDeveloper(
        d,
        achs.filter((a) => a.developerId === d.id),
        pcMap.get(d.id) ?? 0,
      ),
    )
    .sort((a, b) => b.reputation - a.reputation || a.name.localeCompare(b.name));
}

export async function getDeveloperByHandle(handle: string): Promise<DeveloperDTO | null> {
  const all = await getDevelopers();
  return all.find((d) => d.handle === handle.toLowerCase()) ?? null;
}

export async function getAchievements(developerId: number): Promise<AchievementDTO[]> {
  await ensureSeeded();
  const rows = await db
    .select({ a: achievements, hSlug: hackathons.slug })
    .from(achievements)
    .leftJoin(hackathons, eq(achievements.hackathonId, hackathons.id))
    .where(eq(achievements.developerId, developerId))
    .orderBy(desc(achievements.awardedAt));
  return rows.map(({ a, hSlug }) => ({
    id: a.id,
    developerId: a.developerId,
    kind: a.kind as AchievementKind,
    title: a.title,
    issuer: a.issuer,
    place: a.place,
    txHash: a.txHash,
    verified: a.verified,
    date: a.awardedAt.toISOString(),
    hackathonSlug: hSlug,
  }));
}

export async function getAchievementByHash(hash: string) {
  await ensureSeeded();
  const rows = await db
    .select({ a: achievements, name: developers.name, handle: developers.handle })
    .from(achievements)
    .innerJoin(developers, eq(achievements.developerId, developers.id))
    .where(eq(achievements.txHash, hash.trim().toLowerCase()))
    .limit(1);
  const r = rows[0];
  if (!r) return null;
  return {
    title: r.a.title,
    kind: r.a.kind as AchievementKind,
    place: r.a.place,
    issuer: r.a.issuer,
    txHash: r.a.txHash,
    date: r.a.awardedAt.toISOString(),
    holderName: r.name,
    holderHandle: r.handle,
    verified: r.a.verified,
  };
}

export async function getSampleHashes(limit = 2): Promise<string[]> {
  await ensureSeeded();
  const rows = await db
    .select({ h: achievements.txHash })
    .from(achievements)
    .orderBy(desc(achievements.awardedAt))
    .limit(limit);
  return rows.map((r) => r.h);
}

function projectSelect() {
  return db
    .select({
      p: projects,
      ownerName: developers.name,
      ownerHandle: developers.handle,
      hSlug: hackathons.slug,
      hTitle: hackathons.title,
    })
    .from(projects)
    .leftJoin(developers, eq(projects.ownerId, developers.id))
    .leftJoin(hackathons, eq(projects.hackathonId, hackathons.id));
}

type ProjectRow = Awaited<ReturnType<typeof projectSelect>>[number];

function toProject(r: ProjectRow): ProjectDTO {
  const p = r.p;
  return {
    id: p.id,
    slug: p.slug,
    title: p.title,
    summary: p.summary,
    description: p.description,
    scale: p.scale as ProjectScale,
    category: p.category,
    tags: p.tags,
    repoUrl: p.repoUrl,
    demoUrl: p.demoUrl,
    likes: p.likes,
    ownerId: p.ownerId,
    ownerName: r.ownerName,
    ownerHandle: r.ownerHandle,
    hackathonSlug: r.hSlug,
    hackathonTitle: r.hTitle,
    verified: p.verified,
    txHash: p.txHash,
    createdAt: p.createdAt.toISOString(),
  };
}

export async function getProjects(): Promise<ProjectDTO[]> {
  await ensureSeeded();
  const rows = await projectSelect().orderBy(desc(projects.likes), desc(projects.createdAt));
  return rows.map(toProject);
}

export async function getProjectBySlug(slug: string): Promise<ProjectDTO | null> {
  await ensureSeeded();
  const rows = await projectSelect().where(eq(projects.slug, slug)).limit(1);
  return rows[0] ? toProject(rows[0]) : null;
}

export async function getProjectsByOwner(ownerId: number): Promise<ProjectDTO[]> {
  await ensureSeeded();
  const rows = await projectSelect()
    .where(eq(projects.ownerId, ownerId))
    .orderBy(desc(projects.likes));
  return rows.map(toProject);
}

export async function getProjectsByHackathon(slug: string): Promise<ProjectDTO[]> {
  await ensureSeeded();
  const rows = await projectSelect()
    .where(eq(hackathons.slug, slug))
    .orderBy(desc(projects.likes));
  return rows.map(toProject);
}

function hackathonSelect() {
  return db
    .select({
      h: hackathons,
      organizerHandle: developers.handle,
      participants: sql<number>`(select count(*)::int from registrations r where r.hackathon_id = hackathons.id)`,
    })
    .from(hackathons)
    .leftJoin(developers, eq(hackathons.organizerId, developers.id));
}

type HackRow = Awaited<ReturnType<typeof hackathonSelect>>[number];

function toHackathon(r: HackRow): HackathonDTO {
  const h = r.h;
  return {
    id: h.id,
    slug: h.slug,
    title: h.title,
    description: h.description,
    organizer: h.organizer,
    organizerHandle: r.organizerHandle,
    format: h.format as HackathonFormat,
    location: h.location,
    startsAt: h.startsAt.toISOString(),
    endsAt: h.endsAt.toISOString(),
    prizePool: h.prizePool,
    maxParticipants: h.maxParticipants,
    participants: Number(r.participants),
    tags: h.tags,
    status: computeStatus(h.startsAt, h.endsAt),
    createdAt: h.createdAt.toISOString(),
  };
}

export async function getHackathons(): Promise<HackathonDTO[]> {
  await ensureSeeded();
  const rows = await hackathonSelect();
  const list = rows.map(toHackathon);
  const order = { live: 0, upcoming: 1, past: 2 } as const;
  return list.sort((a, b) => {
    if (order[a.status] !== order[b.status]) return order[a.status] - order[b.status];
    const da = new Date(a.startsAt).getTime();
    const db_ = new Date(b.startsAt).getTime();
    return a.status === "past" ? db_ - da : da - db_;
  });
}

export async function getHackathonBySlug(slug: string): Promise<HackathonDTO | null> {
  await ensureSeeded();
  const rows = await hackathonSelect().where(eq(hackathons.slug, slug)).limit(1);
  return rows[0] ? toHackathon(rows[0]) : null;
}

export async function getParticipants(hackathonId: number): Promise<ParticipantDTO[]> {
  await ensureSeeded();
  const rows = await db
    .select({ r: registrations, handle: developers.handle })
    .from(registrations)
    .leftJoin(developers, eq(registrations.developerId, developers.id))
    .where(eq(registrations.hackathonId, hackathonId))
    .orderBy(desc(registrations.createdAt))
    .limit(60);
  return rows.map(({ r, handle }) => ({
    id: r.id,
    name: r.name,
    teamName: r.teamName,
    experience: r.experience,
    handle,
    createdAt: r.createdAt.toISOString(),
  }));
}

export async function getStats(): Promise<StatsDTO> {
  await ensureSeeded();
  const [d, h, p, a] = await Promise.all([
    db.select({ n: count() }).from(developers),
    db.select({ n: count() }).from(hackathons),
    db.select({ n: count() }).from(projects),
    db.select({ n: count() }).from(achievements),
  ]);
  return {
    developers: Number(d[0]?.n ?? 0),
    hackathons: Number(h[0]?.n ?? 0),
    projects: Number(p[0]?.n ?? 0),
    credentials: Number(a[0]?.n ?? 0),
  };
}
import type { Locale } from "./types";

const en = {
  // nav
  "nav.home": "Home",
  "nav.hackathons": "Hackathons",
  "nav.projects": "Projects",
  "nav.developers": "Developers",
  "nav.verify": "Verify",
  "nav.createProfile": "Create portfolio",
  "nav.hostTournament": "Host a tournament",
  "nav.askAi": "AI assistant",
  "nav.menu": "Menu",

  // hero
  "hero.badge": "Web3 platform for verified developer reputation",
  "hero.title1": "Build a reputation",
  "hero.title2": "that lives on-chain",
  "hero.subtitle":
    "Hackathon wins, achievements and projects are confirmed by organizers and written to the blockchain — a transparent portfolio that employers and communities can trust.",
  "hero.cta1": "Explore hackathons",
  "hero.cta2": "Build my portfolio",
  "hero.cta3": "Talk to AI",
  "hero.typed": "hackathon wins|verified portfolios|on-chain reputation|IT tournaments|real projects",
  "hero.chipVerified": "Verified by organizer",
  "hero.chipWinner": "1st place · Genesis",
  "hero.chipTx": "Written to ledger",

  // stats
  "stats.developers": "Developers",
  "stats.hackathons": "Tournaments",
  "stats.projects": "Projects",
  "stats.credentials": "Verified credentials",

  // home
  "home.hackathonsTitle": "Tournaments & hackathons",
  "home.hackathonsSub": "Compete, build with a team and earn verified credentials.",
  "home.projectsTitle": "Projects to explore",
  "home.projectsSub": "From weekend experiments to large-scale products.",
  "home.devsTitle": "Top developers",
  "home.devsSub": "Ranked by verified on-chain reputation.",
  "home.viewAll": "View all",
  "home.howTitle": "How HackChain works",
  "home.howSub": "From the first commit to a proof nobody can fake.",
  "home.step1t": "Join an event",
  "home.step1d": "Register for a hackathon or IT tournament and build with your team.",
  "home.step2t": "Ship your project",
  "home.step2d": "Publish your work — small tools or large-scale products.",
  "home.step3t": "Organizers verify",
  "home.step3d": "Organizers confirm wins and contributions for every participant.",
  "home.step4t": "Proof on-chain",
  "home.step4d": "Each credential gets a transaction hash that anyone can check.",
  "home.aiTitle": "Meet your AI & voice assistant",
  "home.aiDesc":
    "Ask about tournaments, find developers by skill, or just say it out loud — the voice assistant understands English, Russian and Kazakh.",
  "home.aiChat": "Open AI chat",
  "home.aiVoice": "Start voice mode",
  "home.ctaTitle": "Ready to prove what you can build?",
  "home.ctaDesc": "Create your portfolio in a minute or launch your own IT tournament and gather the community.",

  // common
  "common.search": "Search",
  "common.all": "All",
  "common.upcoming": "Upcoming",
  "common.live": "Live now",
  "common.past": "Past",
  "common.online": "Online",
  "common.offline": "Offline",
  "common.hybrid": "Hybrid",
  "common.large": "Large-scale",
  "common.small": "Small",
  "common.participants": "participants",
  "common.registered": "{n} / {max} registered",
  "common.prize": "Prize pool",
  "common.register": "Register",
  "common.details": "Details",
  "common.by": "by",
  "common.verified": "Verified",
  "common.viewPortfolio": "View portfolio",
  "common.repo": "Source code",
  "common.demo": "Live demo",
  "common.loading": "Loading…",
  "common.noResults": "Nothing found. Try another filter.",
  "common.results": "{n} results",
  "common.skills": "Skills",
  "common.reputation": "Reputation",
  "common.openToWork": "Open to work",
  "common.joined": "Joined",
  "common.location": "Location",
  "common.back": "Back",
  "common.copy": "Copy",
  "common.copied": "Copied!",
  "common.date": "Date",
  "common.sortLikes": "Most liked",
  "common.sortNew": "Newest",
  "common.sortRep": "Top reputation",
  "common.sortSoon": "Starting soon",
  "common.scale": "Scale",
  "common.category": "Category",
  "common.format": "Format",
  "common.status": "Status",
  "common.points": "pts",
  "common.projectsCount": "{n} projects",
  "common.winsCount": "{n} wins",

  // pages
  "page.hackathonsTitle": "Hackathons & IT tournaments",
  "page.hackathonsSub": "Find your next challenge — or host your own and gather the community.",
  "page.projectsTitle": "Projects",
  "page.projectsSub": "Browse small tools and large-scale products built by the community.",
  "page.devsTitle": "Developers",
  "page.devsSub": "Discover talented people, their stories and verified achievements.",
  "search.hackathons": "Search tournaments, tags, cities…",
  "search.projects": "Search projects, tags, authors…",
  "search.devs": "Search by name, skill, city…",

  // categories
  "cat.ai": "AI & ML",
  "cat.web3": "Web3",
  "cat.mobile": "Mobile",
  "cat.web": "Web",
  "cat.security": "Security",
  "cat.game": "Games",
  "cat.data": "Data & IoT",
  "cat.devtools": "Dev tools",
  "cat.social": "Social impact",
  "cat.fintech": "Fintech",

  // achievements
  "kind.win": "Winner",
  "kind.finalist": "Finalist",
  "kind.participation": "Participant",
  "kind.project": "Project",
  "kind.certificate": "Certificate",
  "place.1": "1st place",
  "place.2": "2nd place",
  "place.3": "3rd place",
  "place.n": "Top {n}",

  // developer page
  "dev.about": "About",
  "dev.timeline": "Verified achievements",
  "dev.projectsTitle": "Projects",
  "dev.noAchievements": "No achievements yet — join a hackathon to earn the first one.",
  "dev.noProjects": "No projects published yet.",
  "dev.contact": "Links & contacts",
  "dev.score": "Reputation score",
  "dev.wins": "Wins",
  "dev.verifiedCreds": "Verified credentials",
  "dev.wallet": "Wallet",
  "dev.verifiedBy": "Verified by {issuer}",
  "dev.checkTx": "Check on verify page",

  // project page
  "project.about": "About the project",
  "project.team": "Author",
  "project.fromEvent": "Built at",
  "project.like": "Like",
  "project.liked": "Liked",
  "project.onChain": "Project verified on-chain",

  // hackathon page
  "hack.about": "About the event",
  "hack.info": "Event info",
  "hack.when": "When",
  "hack.where": "Where",
  "hack.organizedBy": "Organized by",
  "hack.participantsTitle": "Participants",
  "hack.noParticipants": "Be the first to register!",
  "hack.spotsLeft": "{n} spots left",
  "hack.projectsTitle": "Projects from this event",
  "hack.statusPast": "This event has finished",
  "hack.shareLink": "Copy link",

  // registration
  "reg.title": "Register for this event",
  "reg.name": "Full name",
  "reg.email": "Email",
  "reg.team": "Team name (optional)",
  "reg.handle": "Portfolio handle (optional)",
  "reg.level": "Experience level",
  "reg.beginner": "Beginner",
  "reg.middle": "Middle",
  "reg.pro": "Pro",
  "reg.btn": "Register now",
  "reg.sending": "Registering…",
  "reg.success": "You're in! See you at the event.",
  "reg.full": "Registration is full",
  "reg.closed": "Registration is closed",
  "reg.already": "This email is already registered.",

  // forms
  "form.profileTitle": "Create your developer portfolio",
  "form.profileSub": "Tell the community who you are. Achievements get added when organizers verify them.",
  "form.name": "Full name",
  "form.handle": "Handle",
  "form.handleHint": "3–30 chars: latin letters, digits, - or _",
  "form.role": "What do you do?",
  "form.rolePlaceholder": "Full-stack developer, ML engineer…",
  "form.bio": "Short bio",
  "form.bioPlaceholder": "Who are you, what do you build, what are you proud of?",
  "form.skills": "Skills",
  "form.skillsHint": "Comma separated: React, Solidity, Python",
  "form.location": "City, country",
  "form.github": "GitHub URL",
  "form.website": "Website",
  "form.telegram": "Telegram @username",
  "form.wallet": "Wallet address (optional)",
  "form.openToWork": "I'm open to work & collaborations",
  "form.createProfile": "Create portfolio",
  "form.projectTitle": "Publish a project",
  "form.projectSub": "Share a small tool or a large-scale product with the community.",
  "form.projectName": "Project name",
  "form.summary": "One-line summary",
  "form.description": "Description",
  "form.scale": "Project scale",
  "form.author": "Author",
  "form.selectAuthor": "Select your portfolio",
  "form.noProfile": "No portfolio yet?",
  "form.createFirst": "Create one first",
  "form.hackathonOpt": "Built at event (optional)",
  "form.none": "— none —",
  "form.category": "Category",
  "form.repoUrl": "Repository URL",
  "form.demoUrl": "Demo URL",
  "form.tags": "Tags",
  "form.tagsHint": "Comma separated",
  "form.publish": "Publish project",
  "form.hackTitle": "Host your IT tournament",
  "form.hackSub": "Create a hackathon or contest. People can register right away and compete.",
  "form.eventName": "Tournament name",
  "form.eventDesc": "Description, rules & tracks",
  "form.organizer": "Organizer (you or your team)",
  "form.format": "Format",
  "form.city": "City / venue / platform",
  "form.start": "Starts",
  "form.end": "Ends",
  "form.prize": "Prize pool",
  "form.prizePlaceholder": "$5,000 + swag",
  "form.max": "Max participants",
  "form.createEvent": "Launch tournament",
  "form.saving": "Saving…",
  "form.required": "Please fill in the required fields.",
  "form.success": "Done! Redirecting…",

  // errors
  "err.generic": "Something went wrong. Please try again.",
  "err.email": "Please enter a valid email.",
  "err.handleTaken": "This handle is already taken.",
  "err.dates": "End date must be after the start date.",

  // verify
  "verify.title": "Verify a credential",
  "verify.sub": "Paste a transaction hash from any portfolio to confirm that an achievement is real.",
  "verify.placeholder": "0x…",
  "verify.btn": "Verify",
  "verify.valid": "Credential is valid and verified",
  "verify.invalid": "No credential found for this hash.",
  "verify.holder": "Holder",
  "verify.issuer": "Issued by",
  "verify.type": "Type",
  "verify.tx": "Transaction hash",
  "verify.example": "Try an example",
  "verify.note": "Demo ledger: hashes are generated by HackChain when organizers confirm results.",

  // assistant
  "ai.title": "HackChain AI",
  "ai.subtitle": "Chat & voice assistant",
  "ai.chat": "Chat",
  "ai.voice": "Voice",
  "ai.placeholder": "Ask about tournaments, projects, developers…",
  "ai.send": "Send",
  "ai.listening": "Listening…",
  "ai.speaking": "Speaking…",
  "ai.thinking": "Thinking…",
  "ai.tapSpeak": "Tap to speak",
  "ai.tapStop": "Tap to stop",
  "ai.unsupported": "Voice input isn't supported in this browser. Please use Chrome, Edge or Safari — or type in the chat.",
  "ai.micDenied": "Microphone access was denied. Allow it in the browser address bar and try again.",
  "ai.noSpeech": "I didn't catch that. Tap and try again.",
  "ai.welcome":
    "Hi! I'm the HackChain assistant. I can find tournaments, projects and developers, explain how verification works and open pages for you.",
  "ai.s1": "Upcoming hackathons",
  "ai.s2": "Top developers",
  "ai.s3": "How do I host a tournament?",
  "ai.s4": "What is HackChain?",
  "ai.clear": "Clear chat",
  "ai.soundOn": "Voice replies on",
  "ai.soundOff": "Voice replies off",
  "ai.voiceHint": "Try: “Show upcoming hackathons” or “Open projects”",
  "ai.error": "Sorry, I couldn't answer right now. Please try again.",
  "ai.stopSpeaking": "Stop",

  // footer
  "footer.tagline": "Verifiable developer reputation, powered by hackathons and the blockchain.",
  "footer.platform": "Platform",
  "footer.community": "Community",
  "footer.rights": "All rights reserved.",
  "footer.languages": "Languages",
} as const;

export type DictKey = keyof typeof en;
type Dict = Record<DictKey, string>;

const ru: Dict = {
  "nav.home": "Главная",
  "nav.hackathons": "Хакатоны",
  "nav.projects": "Проекты",
  "nav.developers": "Разработчики",
  "nav.verify": "Проверка",
  "nav.createProfile": "Создать портфолио",
  "nav.hostTournament": "Создать турнир",
  "nav.askAi": "ИИ-помощник",
  "nav.menu": "Меню",

  "hero.badge": "Web3-платформа проверяемой репутации разработчиков",
  "hero.title1": "Репутация разработчика,",
  "hero.title2": "которая живёт в блокчейне",
  "hero.subtitle":
    "Победы на хакатонах, достижения и проекты подтверждаются организаторами и записываются в блокчейн — прозрачное портфолио, которому доверяют работодатели и сообщества.",
  "hero.cta1": "Смотреть хакатоны",
  "hero.cta2": "Создать портфолио",
  "hero.cta3": "Поговорить с ИИ",
  "hero.typed": "победы на хакатонах|проверенные портфолио|репутация в блокчейне|IT-турниры|реальные проекты",
  "hero.chipVerified": "Подтверждено организатором",
  "hero.chipWinner": "1 место · Genesis",
  "hero.chipTx": "Записано в реестр",

  "stats.developers": "Разработчиков",
  "stats.hackathons": "Турниров",
  "stats.projects": "Проектов",
  "stats.credentials": "Подтверждённых наград",

  "home.hackathonsTitle": "Турниры и хакатоны",
  "home.hackathonsSub": "Соревнуйтесь, создавайте с командой и получайте проверенные награды.",
  "home.projectsTitle": "Проекты для знакомства",
  "home.projectsSub": "От экспериментов выходного дня до масштабных продуктов.",
  "home.devsTitle": "Лучшие разработчики",
  "home.devsSub": "Рейтинг по подтверждённой репутации в блокчейне.",
  "home.viewAll": "Смотреть все",
  "home.howTitle": "Как работает HackChain",
  "home.howSub": "От первого коммита до доказательства, которое нельзя подделать.",
  "home.step1t": "Участвуйте",
  "home.step1d": "Зарегистрируйтесь на хакатон или IT-турнир и создавайте вместе с командой.",
  "home.step2t": "Публикуйте проект",
  "home.step2d": "Покажите свою работу — от небольших утилит до крупных продуктов.",
  "home.step3t": "Организаторы подтверждают",
  "home.step3d": "Организаторы подтверждают победы и вклад каждого участника.",
  "home.step4t": "Доказательство в блокчейне",
  "home.step4d": "Каждая награда получает хэш транзакции, который может проверить любой.",
  "home.aiTitle": "Ваш ИИ и голосовой помощник",
  "home.aiDesc":
    "Спрашивайте о турнирах, ищите разработчиков по навыкам или просто скажите вслух — голосовой помощник понимает русский, казахский и английский.",
  "home.aiChat": "Открыть ИИ-чат",
  "home.aiVoice": "Включить голос",
  "home.ctaTitle": "Готовы доказать, что умеете?",
  "home.ctaDesc": "Создайте портфолио за минуту или запустите свой IT-турнир и соберите сообщество.",

  "common.search": "Поиск",
  "common.all": "Все",
  "common.upcoming": "Скоро",
  "common.live": "Идёт сейчас",
  "common.past": "Прошедшие",
  "common.online": "Онлайн",
  "common.offline": "Офлайн",
  "common.hybrid": "Гибрид",
  "common.large": "Масштабные",
  "common.small": "Небольшие",
  "common.participants": "участников",
  "common.registered": "Зарегистрировано {n} / {max}",
  "common.prize": "Призовой фонд",
  "common.register": "Регистрация",
  "common.details": "Подробнее",
  "common.by": "от",
  "common.verified": "Подтверждено",
  "common.viewPortfolio": "Смотреть портфолио",
  "common.repo": "Исходный код",
  "common.demo": "Демо",
  "common.loading": "Загрузка…",
  "common.noResults": "Ничего не найдено. Попробуйте другой фильтр.",
  "common.results": "Найдено: {n}",
  "common.skills": "Навыки",
  "common.reputation": "Репутация",
  "common.openToWork": "Открыт к работе",
  "common.joined": "С нами с",
  "common.location": "Город",
  "common.back": "Назад",
  "common.copy": "Копировать",
  "common.copied": "Скопировано!",
  "common.date": "Дата",
  "common.sortLikes": "Популярные",
  "common.sortNew": "Новые",
  "common.sortRep": "По репутации",
  "common.sortSoon": "Скоро начнутся",
  "common.scale": "Масштаб",
  "common.category": "Категория",
  "common.format": "Формат",
  "common.status": "Статус",
  "common.points": "очк.",
  "common.projectsCount": "Проектов: {n}",
  "common.winsCount": "Побед: {n}",

  "page.hackathonsTitle": "Хакатоны и IT-турниры",
  "page.hackathonsSub": "Найдите новый вызов — или создайте свой турнир и соберите сообщество.",
  "page.projectsTitle": "Проекты",
  "page.projectsSub": "Небольшие утилиты и масштабные продукты, созданные сообществом.",
  "page.devsTitle": "Разработчики",
  "page.devsSub": "Находите талантливых людей, их истории и подтверждённые достижения.",
  "search.hackathons": "Поиск турниров, тегов, городов…",
  "search.projects": "Поиск проектов, тегов, авторов…",
  "search.devs": "Поиск по имени, навыку, городу…",

  "cat.ai": "ИИ и ML",
  "cat.web3": "Web3",
  "cat.mobile": "Мобильные",
  "cat.web": "Веб",
  "cat.security": "Безопасность",
  "cat.game": "Игры",
  "cat.data": "Данные и IoT",
  "cat.devtools": "Dev-инструменты",
  "cat.social": "Социальные",
  "cat.fintech": "Финтех",

  "kind.win": "Победитель",
  "kind.finalist": "Финалист",
  "kind.participation": "Участник",
  "kind.project": "Проект",
  "kind.certificate": "Сертификат",
  "place.1": "1 место",
  "place.2": "2 место",
  "place.3": "3 место",
  "place.n": "Топ-{n}",

  "dev.about": "О разработчике",
  "dev.timeline": "Подтверждённые достижения",
  "dev.projectsTitle": "Проекты",
  "dev.noAchievements": "Пока нет достижений — участвуйте в хакатонах, чтобы получить первое.",
  "dev.noProjects": "Проекты ещё не опубликованы.",
  "dev.contact": "Ссылки и контакты",
  "dev.score": "Рейтинг репутации",
  "dev.wins": "Побед",
  "dev.verifiedCreds": "Подтверждённых наград",
  "dev.wallet": "Кошелёк",
  "dev.verifiedBy": "Подтвердил: {issuer}",
  "dev.checkTx": "Проверить на странице проверки",

  "project.about": "О проекте",
  "project.team": "Автор",
  "project.fromEvent": "Создан на",
  "project.like": "Нравится",
  "project.liked": "Вам нравится",
  "project.onChain": "Проект подтверждён в блокчейне",

  "hack.about": "О мероприятии",
  "hack.info": "Информация",
  "hack.when": "Когда",
  "hack.where": "Где",
  "hack.organizedBy": "Организатор",
  "hack.participantsTitle": "Участники",
  "hack.noParticipants": "Станьте первым участником!",
  "hack.spotsLeft": "Осталось мест: {n}",
  "hack.projectsTitle": "Проекты с этого события",
  "hack.statusPast": "Мероприятие завершено",
  "hack.shareLink": "Копировать ссылку",

  "reg.title": "Регистрация на событие",
  "reg.name": "Имя и фамилия",
  "reg.email": "Email",
  "reg.team": "Название команды (необязательно)",
  "reg.handle": "Ник в портфолио (необязательно)",
  "reg.level": "Уровень опыта",
  "reg.beginner": "Начинающий",
  "reg.middle": "Средний",
  "reg.pro": "Профи",
  "reg.btn": "Зарегистрироваться",
  "reg.sending": "Регистрируем…",
  "reg.success": "Вы в деле! До встречи на событии.",
  "reg.full": "Все места заняты",
  "reg.closed": "Регистрация закрыта",
  "reg.already": "Этот email уже зарегистрирован.",

  "form.profileTitle": "Создайте портфолио разработчика",
  "form.profileSub": "Расскажите сообществу о себе. Достижения появятся после подтверждения организаторами.",
  "form.name": "Имя и фамилия",
  "form.handle": "Никнейм",
  "form.handleHint": "3–30 символов: латиница, цифры, - или _",
  "form.role": "Чем вы занимаетесь?",
  "form.rolePlaceholder": "Full-stack разработчик, ML-инженер…",
  "form.bio": "Коротко о себе",
  "form.bioPlaceholder": "Кто вы, что создаёте и чем гордитесь?",
  "form.skills": "Навыки",
  "form.skillsHint": "Через запятую: React, Solidity, Python",
  "form.location": "Город, страна",
  "form.github": "Ссылка на GitHub",
  "form.website": "Сайт",
  "form.telegram": "Telegram @username",
  "form.wallet": "Адрес кошелька (необязательно)",
  "form.openToWork": "Открыт к работе и сотрудничеству",
  "form.createProfile": "Создать портфолио",
  "form.projectTitle": "Опубликовать проект",
  "form.projectSub": "Поделитесь небольшой утилитой или масштабным продуктом с сообществом.",
  "form.projectName": "Название проекта",
  "form.summary": "Краткое описание",
  "form.description": "Описание",
  "form.scale": "Масштаб проекта",
  "form.author": "Автор",
  "form.selectAuthor": "Выберите своё портфолио",
  "form.noProfile": "Ещё нет портфолио?",
  "form.createFirst": "Сначала создайте",
  "form.hackathonOpt": "Создан на событии (необязательно)",
  "form.none": "— нет —",
  "form.category": "Категория",
  "form.repoUrl": "Ссылка на репозиторий",
  "form.demoUrl": "Ссылка на демо",
  "form.tags": "Теги",
  "form.tagsHint": "Через запятую",
  "form.publish": "Опубликовать проект",
  "form.hackTitle": "Создайте свой IT-турнир",
  "form.hackSub": "Запустите хакатон или конкурс. Люди смогут сразу зарегистрироваться и соревноваться.",
  "form.eventName": "Название турнира",
  "form.eventDesc": "Описание, правила и треки",
  "form.organizer": "Организатор (вы или команда)",
  "form.format": "Формат",
  "form.city": "Город / площадка / платформа",
  "form.start": "Начало",
  "form.end": "Окончание",
  "form.prize": "Призовой фонд",
  "form.prizePlaceholder": "$5 000 + мерч",
  "form.max": "Макс. участников",
  "form.createEvent": "Запустить турнир",
  "form.saving": "Сохраняем…",
  "form.required": "Заполните обязательные поля.",
  "form.success": "Готово! Перенаправляем…",

  "err.generic": "Что-то пошло не так. Попробуйте ещё раз.",
  "err.email": "Введите корректный email.",
  "err.handleTaken": "Этот никнейм уже занят.",
  "err.dates": "Дата окончания должна быть позже даты начала.",

  "verify.title": "Проверка награды",
  "verify.sub": "Вставьте хэш транзакции из любого портфолио, чтобы убедиться, что достижение настоящее.",
  "verify.placeholder": "0x…",
  "verify.btn": "Проверить",
  "verify.valid": "Награда подлинная и подтверждена",
  "verify.invalid": "По этому хэшу ничего не найдено.",
  "verify.holder": "Владелец",
  "verify.issuer": "Выдал",
  "verify.type": "Тип",
  "verify.tx": "Хэш транзакции",
  "verify.example": "Попробовать пример",
  "verify.note": "Демо-реестр: хэши создаёт HackChain, когда организаторы подтверждают результаты.",

  "ai.title": "HackChain AI",
  "ai.subtitle": "Чат и голосовой помощник",
  "ai.chat": "Чат",
  "ai.voice": "Голос",
  "ai.placeholder": "Спросите о турнирах, проектах, разработчиках…",
  "ai.send": "Отправить",
  "ai.listening": "Слушаю…",
  "ai.speaking": "Говорю…",
  "ai.thinking": "Думаю…",
  "ai.tapSpeak": "Нажмите и говорите",
  "ai.tapStop": "Нажмите, чтобы остановить",
  "ai.unsupported": "Голосовой ввод не поддерживается в этом браузере. Используйте Chrome, Edge или Safari — или пишите в чат.",
  "ai.micDenied": "Доступ к микрофону запрещён. Разрешите его в адресной строке браузера и повторите.",
  "ai.noSpeech": "Не расслышал. Нажмите и попробуйте ещё раз.",
  "ai.welcome":
    "Привет! Я ассистент HackChain. Помогу найти турниры, проекты и разработчиков, объясню, как работает подтверждение, и открою нужные страницы.",
  "ai.s1": "Ближайшие хакатоны",
  "ai.s2": "Лучшие разработчики",
  "ai.s3": "Как создать турнир?",
  "ai.s4": "Что такое HackChain?",
  "ai.clear": "Очистить чат",
  "ai.soundOn": "Голосовые ответы включены",
  "ai.soundOff": "Голосовые ответы выключены",
  "ai.voiceHint": "Скажите: «Покажи ближайшие хакатоны» или «Открой проекты»",
  "ai.error": "Извините, сейчас не получилось ответить. Попробуйте ещё раз.",
  "ai.stopSpeaking": "Стоп",

  "footer.tagline": "Проверяемая репутация разработчиков на основе хакатонов и блокчейна.",
  "footer.platform": "Платформа",
  "footer.community": "Сообщество",
  "footer.rights": "Все права защищены.",
  "footer.languages": "Языки",
};

const kk: Dict = {
  "nav.home": "Басты бет",
  "nav.hackathons": "Хакатондар",
  "nav.projects": "Жобалар",
  "nav.developers": "Әзірлеушілер",
  "nav.verify": "Тексеру",
  "nav.createProfile": "Портфолио құру",
  "nav.hostTournament": "Турнир ұйымдастыру",
  "nav.askAi": "ЖИ көмекші",
  "nav.menu": "Мәзір",

  "hero.badge": "Әзірлеушілердің тексерілген беделіне арналған Web3-платформа",
  "hero.title1": "Әзірлеуші беделі",
  "hero.title2": "блокчейнде сақталады",
  "hero.subtitle":
    "Хакатондағы жеңістер, жетістіктер мен жобалар ұйымдастырушылармен расталып, блокчейнге жазылады — жұмыс берушілер мен қауымдастықтар сенетін ашық портфолио.",
  "hero.cta1": "Хакатондарды көру",
  "hero.cta2": "Портфолио құру",
  "hero.cta3": "ЖИ-мен сөйлесу",
  "hero.typed": "хакатондағы жеңістер|тексерілген портфолио|блокчейндегі бедел|IT-турнирлер|нақты жобалар",
  "hero.chipVerified": "Ұйымдастырушы растады",
  "hero.chipWinner": "1 орын · Genesis",
  "hero.chipTx": "Тізілімге жазылды",

  "stats.developers": "Әзірлеуші",
  "stats.hackathons": "Турнир",
  "stats.projects": "Жоба",
  "stats.credentials": "Расталған марапат",

  "home.hackathonsTitle": "Турнирлер мен хакатондар",
  "home.hackathonsSub": "Жарысыңыз, командамен құрыңыз және расталған марапаттар алыңыз.",
  "home.projectsTitle": "Танысуға арналған жобалар",
  "home.projectsSub": "Демалыс күнгі тәжірибелерден ауқымды өнімдерге дейін.",
  "home.devsTitle": "Үздік әзірлеушілер",
  "home.devsSub": "Блокчейндегі расталған бедел бойынша рейтинг.",
  "home.viewAll": "Барлығын көру",
  "home.howTitle": "HackChain қалай жұмыс істейді",
  "home.howSub": "Алғашқы коммиттен жалған жасау мүмкін емес дәлелге дейін.",
  "home.step1t": "Қатысыңыз",
  "home.step1d": "Хакатонға немесе IT-турнирге тіркеліп, командаңызбен бірге жасаңыз.",
  "home.step2t": "Жобаңызды жариялаңыз",
  "home.step2d": "Жұмысыңызды көрсетіңіз — шағын құралдардан ірі өнімдерге дейін.",
  "home.step3t": "Ұйымдастырушылар растайды",
  "home.step3d": "Ұйымдастырушылар әр қатысушының жеңісі мен үлесін растайды.",
  "home.step4t": "Блокчейндегі дәлел",
  "home.step4d": "Әр марапатқа кез келген адам тексере алатын транзакция хэші беріледі.",
  "home.aiTitle": "ЖИ және дауыстық көмекшіңіз",
  "home.aiDesc":
    "Турнирлер туралы сұраңыз, дағдылары бойынша әзірлеуші іздеңіз немесе дауыстап айтыңыз — дауыстық көмекші қазақ, орыс және ағылшын тілдерін түсінеді.",
  "home.aiChat": "ЖИ-чатты ашу",
  "home.aiVoice": "Дауыс режимі",
  "home.ctaTitle": "Қолыңыздан келетінін дәлелдеуге дайынсыз ба?",
  "home.ctaDesc": "Бір минутта портфолио құрыңыз немесе өз IT-турниріңізді ашып, қауымдастықты жинаңыз.",

  "common.search": "Іздеу",
  "common.all": "Барлығы",
  "common.upcoming": "Жақында",
  "common.live": "Қазір өтуде",
  "common.past": "Аяқталған",
  "common.online": "Онлайн",
  "common.offline": "Офлайн",
  "common.hybrid": "Гибрид",
  "common.large": "Ауқымды",
  "common.small": "Шағын",
  "common.participants": "қатысушы",
  "common.registered": "Тіркелді: {n} / {max}",
  "common.prize": "Жүлде қоры",
  "common.register": "Тіркелу",
  "common.details": "Толығырақ",
  "common.by": "автор:",
  "common.verified": "Расталған",
  "common.viewPortfolio": "Портфолионы көру",
  "common.repo": "Бастапқы код",
  "common.demo": "Демо",
  "common.loading": "Жүктелуде…",
  "common.noResults": "Ештеңе табылмады. Басқа сүзгіні таңдаңыз.",
  "common.results": "Табылды: {n}",
  "common.skills": "Дағдылар",
  "common.reputation": "Бедел",
  "common.openToWork": "Жұмысқа ашық",
  "common.joined": "Қосылған күні",
  "common.location": "Қала",
  "common.back": "Артқа",
  "common.copy": "Көшіру",
  "common.copied": "Көшірілді!",
  "common.date": "Күні",
  "common.sortLikes": "Танымал",
  "common.sortNew": "Жаңа",
  "common.sortRep": "Бедел бойынша",
  "common.sortSoon": "Жақында басталады",
  "common.scale": "Ауқымы",
  "common.category": "Санат",
  "common.format": "Формат",
  "common.status": "Күйі",
  "common.points": "ұпай",
  "common.projectsCount": "Жоба: {n}",
  "common.winsCount": "Жеңіс: {n}",

  "page.hackathonsTitle": "Хакатондар мен IT-турнирлер",
  "page.hackathonsSub": "Жаңа сынақ табыңыз — немесе өз турниріңізді ашып, қауымдастықты жинаңыз.",
  "page.projectsTitle": "Жобалар",
  "page.projectsSub": "Қауымдастық жасаған шағын құралдар мен ауқымды өнімдер.",
  "page.devsTitle": "Әзірлеушілер",
  "page.devsSub": "Талантты адамдарды, олардың тарихы мен расталған жетістіктерін табыңыз.",
  "search.hackathons": "Турнир, тег, қала бойынша іздеу…",
  "search.projects": "Жоба, тег, автор бойынша іздеу…",
  "search.devs": "Аты, дағдысы, қаласы бойынша іздеу…",

  "cat.ai": "ЖИ және ML",
  "cat.web3": "Web3",
  "cat.mobile": "Мобильді",
  "cat.web": "Веб",
  "cat.security": "Қауіпсіздік",
  "cat.game": "Ойындар",
  "cat.data": "Дерек және IoT",
  "cat.devtools": "Dev-құралдар",
  "cat.social": "Әлеуметтік",
  "cat.fintech": "Финтех",

  "kind.win": "Жеңімпаз",
  "kind.finalist": "Финалист",
  "kind.participation": "Қатысушы",
  "kind.project": "Жоба",
  "kind.certificate": "Сертификат",
  "place.1": "1 орын",
  "place.2": "2 орын",
  "place.3": "3 орын",
  "place.n": "Топ-{n}",

  "dev.about": "Әзірлеуші туралы",
  "dev.timeline": "Расталған жетістіктер",
  "dev.projectsTitle": "Жобалар",
  "dev.noAchievements": "Әзірге жетістік жоқ — біріншісін алу үшін хакатонға қатысыңыз.",
  "dev.noProjects": "Жобалар әлі жарияланбаған.",
  "dev.contact": "Сілтемелер мен байланыс",
  "dev.score": "Бедел рейтингі",
  "dev.wins": "Жеңіс",
  "dev.verifiedCreds": "Расталған марапат",
  "dev.wallet": "Әмиян",
  "dev.verifiedBy": "Растаған: {issuer}",
  "dev.checkTx": "Тексеру бетінде тексеру",

  "project.about": "Жоба туралы",
  "project.team": "Автор",
  "project.fromEvent": "Жасалған іс-шара",
  "project.like": "Ұнайды",
  "project.liked": "Ұнады",
  "project.onChain": "Жоба блокчейнде расталған",

  "hack.about": "Іс-шара туралы",
  "hack.info": "Ақпарат",
  "hack.when": "Қашан",
  "hack.where": "Қайда",
  "hack.organizedBy": "Ұйымдастырушы",
  "hack.participantsTitle": "Қатысушылар",
  "hack.noParticipants": "Бірінші болып тіркеліңіз!",
  "hack.spotsLeft": "Бос орын: {n}",
  "hack.projectsTitle": "Осы іс-шарадағы жобалар",
  "hack.statusPast": "Іс-шара аяқталды",
  "hack.shareLink": "Сілтемені көшіру",

  "reg.title": "Іс-шараға тіркелу",
  "reg.name": "Аты-жөні",
  "reg.email": "Email",
  "reg.team": "Команда атауы (міндетті емес)",
  "reg.handle": "Портфолио никнеймі (міндетті емес)",
  "reg.level": "Тәжірибе деңгейі",
  "reg.beginner": "Бастаушы",
  "reg.middle": "Орташа",
  "reg.pro": "Кәсіби",
  "reg.btn": "Тіркелу",
  "reg.sending": "Тіркелуде…",
  "reg.success": "Сіз тіркелдіңіз! Іс-шарада кездескенше.",
  "reg.full": "Орындар толды",
  "reg.closed": "Тіркеу жабылды",
  "reg.already": "Бұл email бұрын тіркелген.",

  "form.profileTitle": "Әзірлеуші портфолиосын жасаңыз",
  "form.profileSub": "Қауымдастыққа өзіңіз туралы айтыңыз. Жетістіктер ұйымдастырушылар растағаннан кейін қосылады.",
  "form.name": "Аты-жөні",
  "form.handle": "Никнейм",
  "form.handleHint": "3–30 таңба: латын әріптері, сандар, - немесе _",
  "form.role": "Немен айналысасыз?",
  "form.rolePlaceholder": "Full-stack әзірлеуші, ML-инженер…",
  "form.bio": "Өзіңіз туралы қысқаша",
  "form.bioPlaceholder": "Сіз кімсіз, нені жасайсыз, немен мақтанасыз?",
  "form.skills": "Дағдылар",
  "form.skillsHint": "Үтір арқылы: React, Solidity, Python",
  "form.location": "Қала, ел",
  "form.github": "GitHub сілтемесі",
  "form.website": "Сайт",
  "form.telegram": "Telegram @username",
  "form.wallet": "Әмиян мекенжайы (міндетті емес)",
  "form.openToWork": "Жұмысқа және ынтымақтастыққа ашықпын",
  "form.createProfile": "Портфолио құру",
  "form.projectTitle": "Жобаны жариялау",
  "form.projectSub": "Шағын құралмен немесе ауқымды өніммен қауымдастықпен бөлісіңіз.",
  "form.projectName": "Жоба атауы",
  "form.summary": "Қысқаша сипаттама",
  "form.description": "Сипаттама",
  "form.scale": "Жоба ауқымы",
  "form.author": "Автор",
  "form.selectAuthor": "Портфолиоңызды таңдаңыз",
  "form.noProfile": "Портфолио әлі жоқ па?",
  "form.createFirst": "Алдымен құрыңыз",
  "form.hackathonOpt": "Жасалған іс-шара (міндетті емес)",
  "form.none": "— жоқ —",
  "form.category": "Санат",
  "form.repoUrl": "Репозиторий сілтемесі",
  "form.demoUrl": "Демо сілтемесі",
  "form.tags": "Тегтер",
  "form.tagsHint": "Үтір арқылы",
  "form.publish": "Жобаны жариялау",
  "form.hackTitle": "Өз IT-турниріңізді ұйымдастырыңыз",
  "form.hackSub": "Хакатон немесе байқау ашыңыз. Адамдар бірден тіркеліп, жарыса алады.",
  "form.eventName": "Турнир атауы",
  "form.eventDesc": "Сипаттама, ережелер және тректер",
  "form.organizer": "Ұйымдастырушы (сіз немесе команда)",
  "form.format": "Формат",
  "form.city": "Қала / алаң / платформа",
  "form.start": "Басталуы",
  "form.end": "Аяқталуы",
  "form.prize": "Жүлде қоры",
  "form.prizePlaceholder": "$5 000 + мерч",
  "form.max": "Қатысушылардың ең көп саны",
  "form.createEvent": "Турнирді іске қосу",
  "form.saving": "Сақталуда…",
  "form.required": "Міндетті өрістерді толтырыңыз.",
  "form.success": "Дайын! Бағыттаудамыз…",

  "err.generic": "Бір нәрсе дұрыс болмады. Қайталап көріңіз.",
  "err.email": "Дұрыс email енгізіңіз.",
  "err.handleTaken": "Бұл никнейм бос емес.",
  "err.dates": "Аяқталу күні басталу күнінен кейін болуы керек.",

  "verify.title": "Марапатты тексеру",
  "verify.sub": "Жетістіктің нақты екеніне көз жеткізу үшін кез келген портфолиодан транзакция хэшін қойыңыз.",
  "verify.placeholder": "0x…",
  "verify.btn": "Тексеру",
  "verify.valid": "Марапат шынайы және расталған",
  "verify.invalid": "Бұл хэш бойынша ештеңе табылмады.",
  "verify.holder": "Иесі",
  "verify.issuer": "Берген",
  "verify.type": "Түрі",
  "verify.tx": "Транзакция хэші",
  "verify.example": "Мысалды көру",
  "verify.note": "Демо-тізілім: хэштерді ұйымдастырушылар нәтижені растағанда HackChain жасайды.",

  "ai.title": "HackChain AI",
  "ai.subtitle": "Чат және дауыстық көмекші",
  "ai.chat": "Чат",
  "ai.voice": "Дауыс",
  "ai.placeholder": "Турнирлер, жобалар, әзірлеушілер туралы сұраңыз…",
  "ai.send": "Жіберу",
  "ai.listening": "Тыңдап тұрмын…",
  "ai.speaking": "Айтып жатырмын…",
  "ai.thinking": "Ойланып жатырмын…",
  "ai.tapSpeak": "Басып, сөйлеңіз",
  "ai.tapStop": "Тоқтату үшін басыңыз",
  "ai.unsupported": "Бұл браузерде дауыспен енгізу қолжетімсіз. Chrome, Edge немесе Safari қолданыңыз — не чатқа жазыңыз.",
  "ai.micDenied": "Микрофонға рұқсат берілмеді. Браузердің мекенжай жолағында рұқсат беріп, қайталаңыз.",
  "ai.noSpeech": "Естімедім. Басып, қайта көріңіз.",
  "ai.welcome":
    "Сәлем! Мен HackChain көмекшісімін. Турнирлер, жобалар мен әзірлеушілерді табуға көмектесемін, растау қалай жұмыс істейтінін түсіндіремін және керекті беттерді ашамын.",
  "ai.s1": "Жақын хакатондар",
  "ai.s2": "Үздік әзірлеушілер",
  "ai.s3": "Турнирді қалай ұйымдастырамын?",
  "ai.s4": "HackChain деген не?",
  "ai.clear": "Чатты тазалау",
  "ai.soundOn": "Дауыстық жауап қосулы",
  "ai.soundOff": "Дауыстық жауап өшірулі",
  "ai.voiceHint": "Айтыңыз: «Жақын хакатондарды көрсет» немесе «Жобаларды аш»",
  "ai.error": "Кешіріңіз, қазір жауап бере алмадым. Қайталап көріңіз.",
  "ai.stopSpeaking": "Тоқтату",

  "footer.tagline": "Хакатондар мен блокчейнге негізделген әзірлеушілердің тексерілетін беделі.",
  "footer.platform": "Платформа",
  "footer.community": "Қауымдастық",
  "footer.rights": "Барлық құқықтар қорғалған.",
  "footer.languages": "Тілдер",
};

export const DICT: Record<Locale, Dict> = { en: en as Dict, ru, kk };

export function translate(
  locale: Locale,
  key: DictKey,
  vars?: Record<string, string | number>,
): string {
  let s = DICT[locale][key] ?? DICT.en[key] ?? key;
  if (vars) {
    for (const [k, v] of Object.entries(vars)) s = s.replace(new RegExp(`\\{${k}\\}`, "g"), String(v));
  }
  return s;
}

export const LOCALE_META: Record<Locale, { label: string; short: string; speech: string }> = {
  en: { label: "English", short: "EN", speech: "en-US" },
  ru: { label: "Русский", short: "RU", speech: "ru-RU" },
  kk: { label: "Қазақша", short: "KZ", speech: "kk-KZ" },
};
import { cookies, headers } from "next/headers";
import { translate, type DictKey } from "./dict";
import type { Locale } from "./types";

export const LOCALE_COOKIE = "hc_locale";

function isLocale(v: string | undefined): v is Locale {
  return v === "en" || v === "ru" || v === "kk";
}

export async function getLocale(): Promise<Locale> {
  const store = await cookies();
  const c = store.get(LOCALE_COOKIE)?.value;
  if (isLocale(c)) return c;
  const al = ((await headers()).get("accept-language") ?? "").toLowerCase();
  const first = al.split(",")[0]?.trim() ?? "";
  if (first.startsWith("kk")) return "kk";
  if (first.startsWith("ru")) return "ru";
  if (first.startsWith("en")) return "en";
  if (al.includes("kk")) return "kk";
  if (al.includes("ru")) return "ru";
  return "en";
}

export async function getT() {
  const locale = await getLocale();
  return {
    locale,
    t: (key: DictKey, vars?: Record<string, string | number>) => translate(locale, key, vars),
  };
}
import type { I18nText } from "../db/schema";

export type { I18nText };

export type Locale = "en" | "ru" | "kk";
export const LOCALES: Locale[] = ["en", "ru", "kk"];

export type AchievementKind =
  | "win"
  | "finalist"
  | "participation"
  | "project"
  | "certificate";

export type HackathonStatus = "upcoming" | "live" | "past";
export type HackathonFormat = "online" | "offline" | "hybrid";
export type ProjectScale = "large" | "small";

export interface DeveloperDTO {
  id: number;
  handle: string;
  name: string;
  role: I18nText;
  bio: I18nText;
  location: string;
  skills: string[];
  github: string | null;
  website: string | null;
  telegram: string | null;
  wallet: string | null;
  openToWork: boolean;
  reputation: number;
  achievementsCount: number;
  projectsCount: number;
  winsCount: number;
  joinedAt: string;
}

export interface AchievementDTO {
  id: number;
  developerId: number;
  kind: AchievementKind;
  title: I18nText;
  issuer: string;
  place: number | null;
  txHash: string;
  verified: boolean;
  date: string;
  hackathonSlug: string | null;
}

export interface ProjectDTO {
  id: number;
  slug: string;
  title: I18nText;
  summary: I18nText;
  description: I18nText;
  scale: ProjectScale;
  category: string;
  tags: string[];
  repoUrl: string | null;
  demoUrl: string | null;
  likes: number;
  ownerId: number | null;
  ownerName: string | null;
  ownerHandle: string | null;
  hackathonSlug: string | null;
  hackathonTitle: I18nText | null;
  verified: boolean;
  txHash: string | null;
  createdAt: string;
}

export interface HackathonDTO {
  id: number;
  slug: string;
  title: I18nText;
  description: I18nText;
  organizer: string;
  organizerHandle: string | null;
  format: HackathonFormat;
  location: string;
  startsAt: string;
  endsAt: string;
  prizePool: string;
  maxParticipants: number;
  participants: number;
  tags: string[];
  status: HackathonStatus;
  createdAt: string;
}

export interface ParticipantDTO {
  id: number;
  name: string;
  teamName: string;
  experience: string;
  handle: string | null;
  createdAt: string;
}

export interface StatsDTO {
  developers: number;
  hackathons: number;
  projects: number;
  credentials: number;
}

export function achievementPoints(kind: string, place: number | null): number {
  switch (kind) {
    case "win":
      return place === 1 ? 100 : place === 2 ? 70 : place === 3 ? 50 : 40;
    case "finalist":
      return 30;
    case "project":
      return 20;
    case "certificate":
      return 15;
    default:
      return 10;
  }
}

export function pickText(text: I18nText | null | undefined, locale: Locale): string {
  if (!text) return "";
  return text[locale] || text.en || text.ru || text.kk || "";
}

export function computeStatus(startsAt: Date, endsAt: Date, now = new Date()): HackathonStatus {
  if (now < startsAt) return "upcoming";
  if (now > endsAt) return "past";
  return "live";
}
import type { Locale } from "./types";

export const CATEGORY_ICON: Record<string, string> = {
  ai: "🤖",
  web3: "⛓️",
  mobile: "📱",
  web: "🌐",
  security: "🛡️",
  game: "🎮",
  data: "📊",
  devtools: "🛠️",
  social: "🌍",
  fintech: "💳",
};

export const CATEGORIES = ["ai", "web3", "mobile", "web", "security", "game", "data", "devtools", "social", "fintech"] as const;

const COVERS = [
  "linear-gradient(135deg,#13243b 0%,#1f4a7a 55%,#ffd43e 170%)",
  "linear-gradient(135deg,#1a1f4b 0%,#4b3a9b 60%,#ff9bd2 170%)",
  "linear-gradient(135deg,#0f2c3a 0%,#0e6f78 60%,#7dffd1 170%)",
  "linear-gradient(135deg,#2b1a3b 0%,#8a2f6b 60%,#ffb36b 170%)",
  "linear-gradient(135deg,#102a43 0%,#2f5fa8 60%,#9ad0ff 170%)",
  "linear-gradient(135deg,#1d2b1a 0%,#3f7a2f 60%,#e8ff7a 170%)",
];

const AVATARS = [
  ["#ffd43e", "#f5842a"],
  ["#7dd3fc", "#6366f1"],
  ["#86efac", "#14b8a6"],
  ["#f9a8d4", "#a855f7"],
  ["#fde68a", "#ef4444"],
  ["#a5b4fc", "#22d3ee"],
];

export function hashStr(s: string): number {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) >>> 0;
  return h;
}

export function coverFor(seed: string): string {
  return COVERS[hashStr(seed) % COVERS.length];
}

export function avatarGradient(seed: string): string {
  const [a, b] = AVATARS[hashStr(seed) % AVATARS.length];
  return `linear-gradient(135deg, ${a}, ${b})`;
}

export function initials(name: string): string {
  const parts = name.trim().split(/\s+/).filter(Boolean);
  return ((parts[0]?.[0] ?? "") + (parts.length > 1 ? parts[parts.length - 1][0] : "")).toUpperCase() || "?";
}

export function shortHash(h: string, a = 8, b = 6): string {
  return h.length > a + b + 2 ? `${h.slice(0, a)}…${h.slice(-b)}` : h;
}

const TAGS: Record<Locale, string> = { en: "en-GB", ru: "ru-RU", kk: "kk-KZ" };

export function formatDate(iso: string, locale: Locale, withTime = false): string {
  try {
    return new Date(iso).toLocaleString(TAGS[locale], {
      day: "numeric",
      month: "short",
      year: "numeric",
      ...(withTime ? { hour: "2-digit", minute: "2-digit" } : {}),
      timeZone: "UTC",
    });
  } catch {
    return iso.slice(0, 10);
  }
}

export function formatDay(iso: string, locale: Locale): string {
  try {
    return new Date(iso).toLocaleDateString(TAGS[locale], { day: "numeric", month: "short", timeZone: "UTC" });
  } catch {
    return iso.slice(0, 10);
  }
}

export function dateRange(startIso: string, endIso: string, locale: Locale): string {
  const a = formatDay(startIso, locale);
  const b = formatDay(endIso, locale);
  return a === b ? a : `${a} – ${b}`;
}
import { randomBytes } from "crypto";
import type { I18nText } from "./types";

const TRANSLIT: Record<string, string> = {
  а: "a", б: "b", в: "v", г: "g", д: "d", е: "e", ё: "e", ж: "zh", з: "z", и: "i",
  й: "y", к: "k", л: "l", м: "m", н: "n", о: "o", п: "p", р: "r", с: "s", т: "t",
  у: "u", ф: "f", х: "h", ц: "ts", ч: "ch", ш: "sh", щ: "sch", ъ: "", ы: "y", ь: "",
  э: "e", ю: "yu", я: "ya", ә: "a", ғ: "g", қ: "k", ң: "n", ө: "o", ұ: "u", ү: "u",
  һ: "h", і: "i",
};

export function slugify(input: string): string {
  const lower = input.toLowerCase();
  let out = "";
  for (const ch of lower) out += TRANSLIT[ch] ?? ch;
  return out
    .normalize("NFKD")
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "")
    .slice(0, 40);
}

export function randomSuffix(len = 4): string {
  return randomBytes(len).toString("hex").slice(0, len);
}

export function makeSlug(input: string, fallback = "item"): string {
  const base = slugify(input) || fallback;
  return `${base}-${randomSuffix(4)}`;
}

export function fakeTxHash(): string {
  return "0x" + randomBytes(32).toString("hex");
}

/** Clean user supplied string: trim and clamp. */
export function cleanStr(value: unknown, max = 200): string {
  if (typeof value !== "string") return "";
  return value.trim().replace(/\s+/g, " ").slice(0, max);
}

export function cleanMultiline(value: unknown, max = 3000): string {
  if (typeof value !== "string") return "";
  return value.trim().slice(0, max);
}

export function cleanUrl(value: unknown): string | null {
  const s = cleanStr(value, 300);
  if (!s) return null;
  const withProto = /^https?:\/\//i.test(s) ? s : `https://${s}`;
  try {
    const u = new URL(withProto);
    if (!u.hostname.includes(".")) return null;
    return u.toString();
  } catch {
    return null;
  }
}

export function cleanList(value: unknown, maxItems = 12, maxLen = 30): string[] {
  let arr: string[] = [];
  if (Array.isArray(value)) arr = value.map((v) => cleanStr(v, maxLen));
  else if (typeof value === "string") arr = value.split(/[,;\n]/).map((v) => cleanStr(v, maxLen));
  const seen = new Set<string>();
  const out: string[] = [];
  for (const item of arr) {
    const key = item.toLowerCase();
    if (!item || seen.has(key)) continue;
    seen.add(key);
    out.push(item);
    if (out.length >= maxItems) break;
  }
  return out;
}

/** User writes content in one language: replicate to all three. */
export function sameText(text: string): I18nText {
  return { en: text, ru: text, kk: text };
}

export function isEmail(value: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/.test(value);
}
{
  "dialect": "postgresql",
  "schema": "./src/db/schema.ts",
  "dbCredentials": {
    "url": "postgresql://postgres:postgres@127.0.0.1:5432/app_db"
  }
}
import { defineConfig, globalIgnores } from "eslint/config";
import nextCoreWebVitals from "eslint-config-next/core-web-vitals";

export default defineConfig([
  // Keep the starter on the flat config export that actually runs under the pinned ESLint/Next toolchain.
  ...nextCoreWebVitals,
  globalIgnores([".next/**", "out/**", "build/**", "next-env.d.ts"]),
]);
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
{
  "name": "nextjs-postgresql-template",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "dotenv": "17.3.1",
    "drizzle-orm": "0.45.2",
    "lucide-react": "^1.53.0",
    "next": "16.2.6",
    "pg": "8.20.0",
    "react": "19.2.6",
    "react-dom": "19.2.6"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "4.1.17",
    "@types/node": "22.19.15",
    "@types/pg": "8.18.0",
    "@types/react": "19.2.14",
    "@types/react-dom": "19.2.3",
    "drizzle-kit": "0.31.10",
    "eslint": "9.39.4",
    "eslint-config-next": "16.2.6",
    "postcss": "8.5.8",
    "tailwindcss": "4.1.17",
    "typescript": "5.9.3"
  }
}
const postcssConfig = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default postcssConfig;
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": [
      "dom",
      "dom.iterable",
      "esnext"
    ],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "baseUrl": ".",
    "paths": {
      "@/*": [
        "./src/*"
      ]
    },
    "plugins": [
      {
        "name": "next"
      }
    ]
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts"
  ],
  "exclude": [
    "node_modules"
  ]
}
