# asd
npm install cheerio
export type NovelStatus = 'unread' | 'reading' | 'finished';

export type Novel = {
  id?: string;
  title: string;
  url: string;
  siteName: string;
  bookmark: string;
  status: NovelStatus;
};

const STORAGE_KEY = 'web-novel-manager';

export function getLocalNovels(): Novel[] {
  if (typeof window === 'undefined') return [];
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch {
    return [];
  }
}

export function saveLocalNovels(novels: Novel[]) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(novels));
}

export function addLocalNovel(novel: Novel) {
  const current = getLocalNovels();
  const next = [novel, ...current];
  saveLocalNovels(next);
}

export function removeLocalNovel(url: string) {
  const current = getLocalNovels().filter((n) => n.url !== url);
  saveLocalNovels(current);
}

export function updateLocalNovelStatus(url: string, status: NovelStatus) {
  const current = getLocalNovels().map((n) =>
    n.url === url ? { ...n, status } : n
  );
  saveLocalNovels(current);
}

export function updateLocalNovelBookmark(url: string, bookmark: string) {
  const current = getLocalNovels().map((n) =>
    n.url === url ? { ...n, bookmark } : n
  );
  saveLocalNovels(current);
}'use client';

import { useState } from 'react';
import { Novel } from '@/lib/storage';
import { addLocalNovel, getLocalNovels } from '@/lib/storage';

type Props = {
  onAdd: (novel: Novel) => void;
};

export default function AddNovelForm({ onAdd }: Props) {
  const [url, setUrl] = useState('');
  const [title, setTitle] = useState('');
  const [siteName, setSiteName] = useState('不明');
  const [bookmark, setBookmark] = useState('0');
  const [loading, setLoading] = useState(false);

  async function autoFetchMeta() {
    if (!url.trim()) return;

    setLoading(true);

    try {
      const res = await fetch('/api/scrape', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      const data = await res.json();

      if (!res.ok) {
        throw new Error(data.error || '取得に失敗しました');
      }

      setTitle(data.title || 'タイトル不明');
      setSiteName(data.siteName || '不明');
    } catch (error) {
      console.error(error);
      alert('タイトルの自動取得に失敗しました。手入力してください。');
    } finally {
      setLoading(false);
    }
  }

  function handleSubmit() {
    if (!url.trim() || !title.trim()) {
      alert('URLとタイトルを入力してください');
      return;
    }

    const novel: Novel = {
      title,
      url,
      siteName,
      bookmark,
      status: 'unread',
    };

    const exists = getLocalNovels().some((item) => item.url === novel.url);

    if (exists) {
      alert('すでに登録済みの作品です');
      return;
    }

    addLocalNovel(novel);
    onAdd(novel);

    setUrl('');
    setTitle('');
    setSiteName('不明');
    setBookmark('0');
  }

  return (
    <div className="panel">
      <h2 className="panel-title">小説を追加</h2>

      <div className="form-stack">
        <div className="field">
          <label>URL</label>
          <input
            value={url}
            onChange={(e) => setUrl(e.target.value)}
            placeholder="https://ncode.syosetu.com/..."
          />
        </div>

        <button
          type="button"
          className="secondary-button"
          onClick={autoFetchMeta}
          disabled={loading || !url.trim()}
        >
          {loading ? '取得中...' : 'タイトルを自動取得'}
        </button>

        <div className="field">
          <label>タイトル</label>
          <input
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            placeholder="作品タイトル"
          />
        </div>

        <div className="field">
          <label>しおり</label>
          <input
            value={bookmark}
            onChange={(e) => setBookmark(e.target.value)}
            placeholder="第12話 / 35節"
          />
        </div>

        <button type="button" className="primary-button" onClick={handleSubmit}>
          追加する
        </button>
      </div>
    </div>
  );
}import { Novel } from '@/lib/storage';

type Props = {
  novel: Novel;
  onStatusChange: (url: string, status: Novel['status']) => void;
  onBookmarkChange: (url: string, bookmark: string) => void;
  onDelete: (url: string) => void;
};

const statusStyles: Record<Novel['status'], string> = {
  unread: 'status-badge unread',
  reading: 'status-badge reading',
  finished: 'status-badge finished',
};

export default function NovelCard({
  novel,
  onStatusChange,
  onBookmarkChange,
  onDelete,
}: Props) {
  return (
    <div className="novel-card">
      <div className="novel-top">
        <div>
          <h3>{novel.title}</h3>
          <p>{novel.siteName}</p>
        </div>

        <span className={statusStyles[novel.status]}>
          {novel.status === 'unread'
            ? '未読'
            : novel.status === 'reading'
              ? '読中'
              : '読了'}
        </span>
      </div>

      <div className="bookmark-box">
        <label>しおり</label>
        <input
          value={novel.bookmark}
          onChange={(e) => onBookmarkChange(novel.url, e.target.value)}
        />
      </div>

      <div className="status-box">
        <label>ステータス</label>
        <div className="status-buttons">
          {(['unread', 'reading', 'finished'] as const).map((status) => (
            <button
              key={status}
              type="button"
              className={novel.status === status ? 'status-button active' : 'status-button'}
              onClick={() => onStatusChange(novel.url, status)}
            >
              {status === 'unread'
                ? '未読'
                : status === 'reading'
                  ? '読中'
                  : '読了'}
            </button>
          ))}
        </div>
      </div>

      <div className="link-row">
        <a href={novel.url} target="_blank" rel="noreferrer">
          作品ページを開く
        </a>
      </div>

      <div className="delete-row">
        <button type="button" className="delete-button" onClick={() => onDelete(novel.url)}>
          削除
        </button>
      </div>
    </div>
  );
}'use client';

import { useEffect, useState } from 'react';
import AddNovelForm from '@/components/AddNovelForm';
import NovelCard from '@/components/NovelCard';
import {
  getLocalNovels,
  removeLocalNovel,
  updateLocalNovelBookmark,
  updateLocalNovelStatus,
  type Novel,
} from '@/lib/storage';

export default function HomePage() {
  const [novels, setNovels] = useState<Novel[]>([]);
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setNovels(getLocalNovels());
    setMounted(true);
  }, []);

  if (!mounted) return null;

  function handleAdd(novel: Novel) {
    setNovels((prev) => [novel, ...prev]);
  }

  function handleStatusChange(url: string, status: Novel['status']) {
    updateLocalNovelStatus(url, status);
    setNovels((prev) =>
      prev.map((n) => (n.url === url ? { ...n, status } : n))
    );
  }

  function handleBookmarkChange(url: string, bookmark: string) {
    updateLocalNovelBookmark(url, bookmark);
    setNovels((prev) =>
      prev.map((n) => (n.url === url ? { ...n, bookmark } : n))
    );
  }

  function handleDelete(url: string) {
    removeLocalNovel(url);
    setNovels((prev) => prev.filter((n) => n.url !== url));
  }

  return (
    <main className="page-shell">
      <div className="container">
        <header className="header">
          <div>
            <p className="eyebrow">Web小説読書管理</p>
            <h1>Web小説 読書管理マネージャー</h1>
          </div>
        </header>

        <div className="layout">
          <AddNovelForm onAdd={handleAdd} />

          <section className="list-area">
            {novels.length === 0 ? (
              <div className="empty-state">
                <h3>まだ作品がありません</h3>
                <p>URLを登録して、作品の読書管理を始めましょう。</p>
              </div>
            ) : (
              novels.map((novel) => (
                <NovelCard
                  key={novel.url}
                  novel={novel}
                  onStatusChange={handleStatusChange}
                  onBookmarkChange={handleBookmarkChange}
                  onDelete={handleDelete}
                />
              ))
            )}
          </section>
        </div>
      </div>
    </main>
  );
}import { NextResponse } from 'next/server';
import * as cheerio from 'cheerio';

export async function POST(request: Request) {
  try {
    const { url } = await request.json();

    if (!url) {
      return NextResponse.json({ error: 'URLが必要です' }, { status: 400 });
    }

    const res = await fetch(url, {
      headers: {
        'User-Agent':
          'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36',
      },
      cache: 'no-store',
    });

    if (!res.ok) {
      return NextResponse.json({ error: 'URLの取得に失敗しました' }, { status: 400 });
    }

    const html = await res.text();
    const $ = cheerio.load(html);

    const title =
      $('meta[property="og:title"]').attr('content') ||
      $('title').text().trim() ||
      'タイトル不明';

    const siteName = detectSite(url);

    return NextResponse.json({
      title: cleanTitle(title),
      siteName,
    });
  } catch (error) {
    return NextResponse.json({ error: 'スクレイピングに失敗しました' }, { status: 500 });
  }
}

function detectSite(url: string) {
  if (url.includes('syosetu.com')) return '小説家になろう';
  if (url.includes('kakuyomu.jp')) return 'カクヨム';
  if (url.includes('alpha-polish.com')) return 'アルファポリス';
  if (url.includes('novelup.plus')) return 'NovelUp';
  return '不明';
}

function cleanTitle(value: string) {
  return value
    .replace(/\s+/g, ' ')
    .replace(/[-|｜].*$/, '')
    .trim();
}html,
body {
  margin: 0;
  padding: 0;
  background: #f5f7fb;
  color: #1f2937;
  font-family: 'Segoe UI', 'Hiragino Sans', 'Noto Sans JP', sans-serif;
}

* {
  box-sizing: border-box;
}

a {
  color: inherit;
  text-decoration: none;
}

button,
input {
  font: inherit;
}

.page-shell {
  min-height: 100vh;
  padding: 40px 18px;
}

.container {
  max-width: 1180px;
  margin: 0 auto;
}

.header {
  margin-bottom: 28px;
}

.eyebrow {
  margin: 0 0 8px;
  color: #10b981;
  font-size: 12px;
  letter-spacing: 0.08em;
  font-weight: 700;
  text-transform: uppercase;
}

.header h1 {
  margin: 0;
  font-size: clamp(2rem, 4vw, 3rem);
  line-height: 1.1;
  color: #0f172a;
}

.layout {
  display: grid;
  grid-template-columns: minmax(300px, 420px) minmax(0, 1fr);
  gap: 24px;
  align-items: start;
}

.panel {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 20px;
  padding: 22px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.04);
}

.panel-title {
  margin: 0 0 18px;
  font-size: 1.3rem;
  color: #111827;
}

.form-stack {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.field label {
  font-size: 0.9rem;
  font-weight: 700;
  color: #374151;
}

.field input {
  width: 100%;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  padding: 12px 13px;
  background: #fff;
  color: #111827;
  outline: none;
}

.field input:focus {
  border-color: #10b981;
  box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.12);
}

.primary-button,
.secondary-button,
.delete-button,
.status-button {
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition: 0.2s ease;
}

.primary-button {
  background: linear-gradient(135deg, #10b981, #059669);
  color: white;
  padding: 13px 14px;
  font-weight: 700;
}

.primary-button:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(16, 185, 129, 0.18);
}

.secondary-button {
  background: #f3f4f6;
  color: #111827;
  padding: 10px 12px;
  font-weight: 600;
}

.secondary-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.list-area {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.empty-state {
  background: #ffffff;
  border: 1px dashed #d1d5db;
  border-radius: 20px;
  padding: 42px 20px;
  text-align: center;
  color: #6b7280;
}

.empty-state h3 {
  margin: 0 0 10px;
  color: #374151;
}

.novel-card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 20px;
  padding: 18px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.04);
}

.novel-top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 14px;
}

.novel-top h3 {
  margin: 0;
  font-size: 1.2rem;
  color: #0f172a;
  line-height: 1.4;
}

.novel-top p {
  margin: 6px 0 0;
  font-size: 0.88rem;
  color: #6b7280;
}

.status-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 999px;
  padding: 8px 12px;
  font-size: 0.78rem;
  font-weight: 700;
  white-space: nowrap;
}

.status-badge.unread {
  background: #f3f4f6;
  color: #374151;
}

.status-badge.reading {
  background: #dcfce7;
  color: #166534;
}

.status-badge.finished {
  background: #dbeafe;
  color: #1d4ed8;
}

.bookmark-box,
.status-box {
  margin-top: 12px;
}

.bookmark-box label,
.status-box label {
  display: block;
  margin-bottom: 8px;
  font-size: 0.84rem;
  font-weight: 700;
  color: #374151;
}

.bookmark-box input {
  width: 100%;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  padding: 11px 12px;
  outline: none;
}

.bookmark-box input:focus {
  border-color: #10b981;
  box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.12);
}

.status-buttons {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.status-button {
  flex: 1 1 0;
  min-width: 70px;
  background: #f3f4f6;
  color: #374151;
  padding: 10px 8px;
  font-weight: 700;
}

.status-button.active {
  background: #10b981;
  color: white;
}

.link-row {
  margin-top: 14px;
}

.link-row a {
  color: #059669;
  font-weight: 700;
  text-decoration: underline;
}

.delete-row {
  margin-top: 14px;
  display: flex;
  justify-content: flex-end;
}

.delete-button {
  background: #fef2f2;
  color: #dc2626;
  border: 1px solid #fecaca;
  padding: 8px 12px;
  font-weight: 700;
}

@media (max-width: 900px) {
  .layout {
    grid-template-columns: 1fr;
  }
}import './globals.css';

export const metadata = {
  title: 'Web小説 読書管理マネージャー',
  description: 'Web小説の読書進捗を管理するアプリ',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}
