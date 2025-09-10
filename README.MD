import { useEffect, useMemo, useState, useRef } from "react";
import ReactMarkdown from "react-markdown";
import {
  Link as LucideLink,
  Youtube,
  Facebook,
  ShoppingBag,
  Mail,
  Phone,
  MessageCircle,
  Globe,
} from "lucide-react";

/**
 * Linktree-like Single-User Page (Upgraded)
 * - Unlimited links
 * - Customizable text layout (headings, paragraphs, alignment, sizes)
 * - 1 or 2 column layout
 * - LocalStorage persistence (no backend)
 * - Simple Admin/Edit Mode (toggle with password prompt; default password: "admin")
 * - Export / Import JSON
 * - Tailwind styles
 * - NEW: Icons for link blocks (emoji, URL image, or selected Lucide icon)
 * - NEW: Drag & Drop reordering (vanilla HTML5 DnD)
 * - NEW: Markdown support for Text blocks (bold, lists, links, etc.)
 *
 * Usage in Next.js:
 * 1) Ensure Tailwind is set up.
 * 2) `npm i react-markdown lucide-react`
 * 3) Save this file as pages/index.jsx
 * 4) npm run dev → open http://localhost:3000
 */

const LS_KEY = "linktree_single_user_v2";
const DEFAULT_PASSWORD = "admin"; // change after first run in Settings

const defaultData = {
  profile: {
    username: "jay",
    displayName: "Jay Kay",
    avatar: "https://i.pravatar.cc/200?img=3",
    bio: "Dev / Runner / Tech Explorer",
    themeColor: "#7c3aed",
    accentColor: "#ec4899",
    linkStyle: "solid", // solid | outline | ghost
    password: DEFAULT_PASSWORD,
  },
  layout: {
    align: "center", // center | left
    columns: 1, // 1 | 2
    width: "md", // sm | md | lg
  },
  blocks: [
    { type: "heading", text: "Follow me", align: "center", size: "xl", weight: "bold" },
    { type: "text", text: "รวมลิงก์ทั้งหมดไว้ที่นี่ ✨

- **Facebook** สำหรับข่าวสาร
- **YouTube** สำหรับวิดีโอ

กดปุ่มด้านล่างได้เลย", align: "center" },
    { type: "link", title: "Facebook", url: "https://facebook.com/yourname", subtitle: "ติดตามข่าวสาร", badge: "Social", variant: "solid", icon: { mode: "lucide", value: "Facebook" } },
    { type: "link", title: "YouTube", url: "https://youtube.com/@yourchannel", subtitle: "วิดีโอล่าสุด", badge: "Video", variant: "outline", icon: { mode: "lucide", value: "Youtube" } },
    { type: "link", title: "Shopee Store", url: "https://shopee.co.th/yourshop", subtitle: "ช้อปสินค้า", badge: "Shop", variant: "ghost", icon: { mode: "lucide", value: "ShoppingBag" } },
    { type: "divider" },
    { type: "heading", text: "Contact", align: "left", size: "lg", weight: "semibold" },
    { type: "text", text: "**Email:** you@example.com

**LINE:** @yourlineid", align: "left" },
  ],
};

export default function LinktreeSingleUser() {
  const [data, setData] = useStickyState(defaultData, LS_KEY);
  const [isEdit, setIsEdit] = useState(false);
  const [showSettings, setShowSettings] = useState(false);
  const [importText, setImportText] = useState("");
  const dragIndexRef = useRef(null);

  const widthClass = useMemo(() => {
    switch (data.layout.width) {
      case "sm":
        return "max-w-sm";
      case "lg":
        return "max-w-3xl";
      default:
        return "max-w-xl";
    }
  }, [data.layout.width]);

  const gridClass = data.layout.columns === 2 ? "grid md:grid-cols-2 gap-4" : "grid grid-cols-1 gap-4";

  const pageAlignClass = data.layout.align === "left" ? "items-start text-left" : "items-center text-center";

  function requestEditToggle() {
    if (isEdit) {
      setIsEdit(false);
      return;
    }
    const answer = typeof window !== "undefined" ? window.prompt("Enter password to edit:") : null;
    if (answer === data.profile.password) setIsEdit(true);
    else if (answer != null) alert("Wrong password");
  }

  function handleExport() {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `linktree-${data.profile.username}.json`;
    a.click();
    URL.revokeObjectURL(url);
  }

  function handleImport() {
    try {
      const parsed = JSON.parse(importText);
      setData(parsed);
      setImportText("");
      setShowSettings(false);
    } catch (e) {
      alert("Invalid JSON");
    }
  }

  // DnD handlers
  function onDragStart(i) {
    dragIndexRef.current = i;
  }
  function onDragOver(e) { e.preventDefault(); }
  function onDrop(i) {
    const from = dragIndexRef.current;
    if (from == null || from === i) return;
    const blocks = [...data.blocks];
    const [moved] = blocks.splice(from, 1);
    blocks.splice(i, 0, moved);
    setData({ ...data, blocks });
    dragIndexRef.current = null;
  }

  return (
    <div
      className="min-h-screen flex flex-col"
      style={{
        background: `linear-gradient(180deg, ${data.profile.themeColor} 0%, ${data.profile.accentColor} 100%)`,
      }}
    >
      <HeaderBar
        isEdit={isEdit}
        onToggleEdit={requestEditToggle}
        onOpenSettings={() => setShowSettings(true)}
        onExport={handleExport}
      />

      <main className={`flex-1 flex ${pageAlignClass} px-4 py-10`}>
        <section className={`${widthClass} w-full`}>
          <ProfileHeader
            data={data}
            onChange={(p) => updateProfile(setData, data, p)}
            isEdit={isEdit}
          />

          {/* Blocks */}
          <div className={`mt-8 ${gridClass}`}>
            {data.blocks.map((b, idx) => (
              <Draggable
                key={idx}
                index={idx}
                isEdit={isEdit}
                onDragStart={() => onDragStart(idx)}
                onDragOver={onDragOver}
                onDrop={() => onDrop(idx)}
              >
                <BlockRenderer
                  block={b}
                  alignGlobal={data.layout.align}
                  linkStyle={data.profile.linkStyle}
                >
                  {isEdit && (
                    <BlockEditor
                      block={b}
                      onChange={(nb) => updateBlock(setData, data, idx, nb)}
                      onDelete={() => deleteBlock(setData, data, idx)}
                      onMoveUp={() => moveBlock(setData, data, idx, -1)}
                      onMoveDown={() => moveBlock(setData, data, idx, 1)}
                    />
                  )}
                </BlockRenderer>
              </Draggable>
            ))}
          </div>

          {isEdit && (
            <AddBlockBar onAdd={(blk) => addBlock(setData, data, blk)} />
          )}
        </section>
      </main>

      {isEdit && (
        <footer className="p-4 text-center text-white/80">
          Saved locally • Use Export to back up your data • Drag blocks by the handle (≡)
        </footer>
      )}

      {showSettings && (
        <SettingsModal
          data={data}
          onClose={() => setShowSettings(false)}
          onSave={(patch) => updateSettings(setData, data, patch)}
          importText={importText}
          setImportText={setImportText}
          onImport={handleImport}
        />
      )}
    </div>
  );
}

/* ------------------------- Hooks & Helpers -------------------------- */
function useStickyState(defaultValue, key) {
  const [value, setValue] = useState(defaultValue);
  useEffect(() => {
    try {
      const stickyValue = window.localStorage.getItem(key);
      if (stickyValue !== null) setValue(JSON.parse(stickyValue));
    } catch (e) {}
  }, [key]);
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (e) {}
  }, [key, value]);
  return [value, setValue];
}

function updateProfile(setData, data, patch) {
  setData({ ...data, profile: { ...data.profile, ...patch } });
}

function updateSettings(setData, data, patch) {
  setData({ ...data, layout: { ...data.layout, ...patch.layout }, profile: { ...data.profile, ...patch.profile } });
}

function updateBlock(setData, data, idx, nb) {
  const blocks = [...data.blocks];
  blocks[idx] = nb;
  setData({ ...data, blocks });
}

function deleteBlock(setData, data, idx) {
  const blocks = data.blocks.filter((_, i) => i !== idx);
  setData({ ...data, blocks });
}

function moveBlock(setData, data, idx, delta) {
  const blocks = [...data.blocks];
  const ni = idx + delta;
  if (ni < 0 || ni >= blocks.length) return;
  const tmp = blocks[idx];
  blocks[idx] = blocks[ni];
  blocks[ni] = tmp;
  setData({ ...data, blocks });
}

function addBlock(setData, data, blk) {
  setData({ ...data, blocks: [...data.blocks, blk] });
}

/* -------------------------- UI Components -------------------------- */
function HeaderBar({ isEdit, onToggleEdit, onOpenSettings, onExport }) {
  return (
    <div className="sticky top-0 z-20 bg-black/10 backdrop-blur border-b border-white/10">
      <div className="max-w-5xl mx-auto px-4 py-3 flex items-center justify-between text-white">
        <div className="font-semibold tracking-wide">Linktree — Single User</div>
        <div className="flex items-center gap-2">
          <button className="btn" onClick={onOpenSettings}>Settings</button>
          <button className="btn" onClick={onExport}>Export JSON</button>
          <button className={`btn ${isEdit ? "!bg-rose-600" : ""}`} onClick={onToggleEdit}>
            {isEdit ? "Done" : "Edit"}
          </button>
        </div>
      </div>
      <style jsx>{`
        .btn { @apply bg-white/15 hover:bg-white/25 px-3 py-1.5 rounded-xl text-sm transition; }
      `}</style>
    </div>
  );
}

function ProfileHeader({ data, onChange, isEdit }) {
  const { profile, layout } = data;
  const alignClass = layout.align === "left" ? "items-start text-left" : "items-center text-center";
  return (
    <div className={`flex flex-col ${alignClass} gap-3`}>
      <img src={profile.avatar} alt={profile.displayName} className="w-28 h-28 rounded-full shadow-xl border-4 border-white/30" />
      {isEdit ? (
        <input
          className="input text-3xl font-bold"
          value={profile.displayName}
          onChange={(e) => onChange({ displayName: e.target.value })}
        />
      ) : (
        <h1 className="text-white text-3xl font-extrabold drop-shadow-sm">{profile.displayName}</h1>
      )}
      {isEdit ? (
        <textarea
          className="input min-h-[60px]"
          value={profile.bio}
          onChange={(e) => onChange({ bio: e.target.value })}
        />
      ) : (
        <p className="text-white/90 whitespace-pre-line">{profile.bio}</p>
      )}
      {isEdit && (
        <div className="flex flex-wrap gap-2 mt-1">
          <label className="label">
            Avatar URL
            <input className="input" value={profile.avatar} onChange={(e) => onChange({ avatar: e.target.value })} />
          </label>
          <label className="label">
            Theme
            <input type="color" className="input !p-1" value={profile.themeColor} onChange={(e) => onChange({ themeColor: e.target.value })} />
          </label>
          <label className="label">
            Accent
            <input type="color" className="input !p-1" value={profile.accentColor} onChange={(e) => onChange({ accentColor: e.target.value })} />
          </label>
          <label className="label">
            Link Style
            <select className="input" value={profile.linkStyle} onChange={(e) => onChange({ linkStyle: e.target.value })}>
              <option value="solid">solid</option>
              <option value="outline">outline</option>
              <option value="ghost">ghost</option>
            </select>
          </label>
        </div>
      )}
      <style jsx>{`
        .label { @apply flex flex-col text-white/90 text-sm; }
        .input { @apply bg-white/90 text-gray-900 rounded-xl px-3 py-2 shadow-sm outline-none focus:ring-2 focus:ring-indigo-400; }
      `}</style>
    </div>
  );
}

function Draggable({ index, isEdit, onDragStart, onDragOver, onDrop, children }) {
  return (
    <div
      draggable={isEdit}
      onDragStart={() => onDragStart(index)}
      onDragOver={onDragOver}
      onDrop={onDrop}
      className={`relative group ${isEdit ? "cursor-grab" : ""}`}
    >
      {isEdit && (
        <div className="absolute -left-3 -top-3 select-none">
          <span className="text-white/70 text-lg">≡</span>
        </div>
      )}
      {children}
    </div>
  );
}

function BlockRenderer({ block, children, alignGlobal, linkStyle }) {
  if (block.type === "heading") {
    const align = clsAlign(block.align ?? alignGlobal);
    const sizeCls = block.size === "xl" ? "text-2xl" : block.size === "lg" ? "text-xl" : "text-lg";
    const weightCls = block.weight === "bold" ? "font-extrabold" : block.weight === "semibold" ? "font-semibold" : "font-medium";
    return (
      <div className={`col-span-full text-white ${align}`}>
        <h2 className={`${sizeCls} ${weightCls} drop-shadow-sm`}>{block.text}</h2>
        {children}
      </div>
    );
  }
  if (block.type === "text") {
    const align = clsAlign(block.align ?? alignGlobal);
    return (
      <div className={`col-span-full text-white/95 ${align}`}>
        <ReactMarkdown
          components={{
            a: (props) => <a {...props} className="underline" target="_blank" rel="noreferrer" />,
          }}
        >
          {block.text || ""}
        </ReactMarkdown>
        {children}
      </div>
    );
  }
  if (block.type === "divider") {
    return (
      <div className="col-span-full">
        <hr className="border-white/30" />
        {children}
      </div>
    );
  }
  if (block.type === "spacer") {
    const h = block.size === "lg" ? "h-10" : block.size === "md" ? "h-6" : "h-3";
    return (
      <div className={`col-span-full ${h}`}>{children}</div>
    );
  }
  if (block.type === "link") {
    const variant = block.variant || linkStyle || "solid";
    return (
      <div>
        <a
          href={block.url}
          target="_blank"
          rel="noopener noreferrer"
          className={linkButtonClass(variant)}
        >
          <div className="flex items-center justify-between gap-3">
            <div className="flex items-center gap-3">
              <LinkIcon icon={block.icon} />
              <div className="text-left">
                <div className="text-base font-semibold">{block.title}</div>
                {block.subtitle && (
                  <div className="text-sm opacity-90">{block.subtitle}</div>
                )}
              </div>
            </div>
            {block.badge && (
              <span className="text-xs px-2 py-1 rounded-full bg-black/20 border border-white/20">
                {block.badge}
              </span>
            )}
          </div>
        </a>
        {children}
      </div>
    );
  }
  return null;
}

function LinkIcon({ icon }) {
  const size = 20;
  if (!icon || !icon.mode || icon.mode === "none") return <LucideLink size={size} className="opacity-90" />;
  if (icon.mode === "emoji") return <span className="text-xl leading-none">{icon.value || "🔗"}</span>;
  if (icon.mode === "url") return <img src={icon.value} alt="icon" width={size} height={size} className="rounded" />;
  if (icon.mode === "lucide") {
    const map = {
      Link: LucideLink,
      Youtube,
      Facebook,
      ShoppingBag,
      Mail,
      Phone,
      MessageCircle,
      Globe,
    };
    const Comp = map[icon.value] || LucideLink;
    return <Comp size={size} className="opacity-90" />;
  }
  return <LucideLink size={size} className="opacity-90" />;
}

function BlockEditor({ block, onChange, onDelete, onMoveUp, onMoveDown }) {
  return (
    <div className="mt-2 bg-white/10 backdrop-blur rounded-xl p-3 border border-white/20 text-white">
      <div className="flex flex-wrap gap-2 mb-2">
        <button className="btn-small" onClick={onMoveUp}>↑</button>
        <button className="btn-small" onClick={onMoveDown}>↓</button>
        <button className="btn-small !bg-rose-600" onClick={onDelete}>Delete</button>
      </div>

      {block.type === "heading" && (
        <div className="grid md:grid-cols-4 gap-2">
          <label className="label">Text<input className="input" value={block.text} onChange={(e) => onChange({ ...block, text: e.target.value })} /></label>
          <label className="label">Align<select className="input" value={block.align || "inherit"} onChange={(e) => onChange({ ...block, align: optInherit(e.target.value) })}><option>inherit</option><option>left</option><option>center</option></select></label>
          <label className="label">Size<select className="input" value={block.size || "lg"} onChange={(e) => onChange({ ...block, size: e.target.value })}><option>lg</option><option>xl</option><option>md</option></select></label>
          <label className="label">Weight<select className="input" value={block.weight || "semibold"} onChange={(e) => onChange({ ...block, weight: e.target.value })}><option>semibold</option><option>bold</option><option>medium</option></select></label>
        </div>
      )}

      {block.type === "text" && (
        <div className="grid md:grid-cols-3 gap-2">
          <label className="label md:col-span-2">Text (Markdown supported)
            <textarea className="input min-h-[120px]" value={block.text} onChange={(e) => onChange({ ...block, text: e.target.value })} />
          </label>
          <label className="label">Align<select className="input" value={block.align || "inherit"} onChange={(e) => onChange({ ...block, align: optInherit(e.target.value) })}><option>inherit</option><option>left</option><option>center</option></select></label>
        </div>
      )}

      {block.type === "link" && (
        <div className="grid md:grid-cols-4 gap-2">
          <label className="label md:col-span-2">Title<input className="input" value={block.title} onChange={(e) => onChange({ ...block, title: e.target.value })} /></label>
          <label className="label md:col-span-2">URL<input className="input" value={block.url} onChange={(e) => onChange({ ...block, url: e.target.value })} /></label>
          <label className="label md:col-span-2">Subtitle<input className="input" value={block.subtitle || ""} onChange={(e) => onChange({ ...block, subtitle: e.target.value })} /></label>
          <label className="label">Badge<input className="input" value={block.badge || ""} onChange={(e) => onChange({ ...block, badge: e.target.value })} /></label>
          <label className="label">Variant<select className="input" value={block.variant || "inherit"} onChange={(e) => onChange({ ...block, variant: optInherit(e.target.value) })}><option>inherit</option><option>solid</option><option>outline</option><option>ghost</option></select></label>

          {/* Icon settings */}
          <div className="md:col-span-4 grid md:grid-cols-4 gap-2">
            <label className="label">Icon Mode
              <select className="input" value={block.icon?.mode || "none"} onChange={(e) => onChange({ ...block, icon: { ...(block.icon || {}), mode: e.target.value } })}>
                <option value="none">none</option>
                <option value="emoji">emoji</option>
                <option value="url">image url</option>
                <option value="lucide">lucide</option>
              </select>
            </label>

            {block.icon?.mode === "emoji" && (
              <label className="label md:col-span-3">Emoji<input className="input" placeholder="e.g. 🔗" value={block.icon?.value || ""} onChange={(e) => onChange({ ...block, icon: { mode: "emoji", value: e.target.value } })} /></label>
            )}
            {block.icon?.mode === "url" && (
              <label className="label md:col-span-3">Image URL<input className="input" placeholder="https://..." value={block.icon?.value || ""} onChange={(e) => onChange({ ...block, icon: { mode: "url", value: e.target.value } })} /></label>
            )}
            {block.icon?.mode === "lucide" && (
              <label className="label md:col-span-3">Lucide Icon
                <select className="input" value={block.icon?.value || "Link"} onChange={(e) => onChange({ ...block, icon: { mode: "lucide", value: e.target.value } })}>
                  <option>Link</option>
                  <option>Youtube</option>
                  <option>Facebook</option>
                  <option>ShoppingBag</option>
                  <option>Mail</option>
                  <option>Phone</option>
                  <option>MessageCircle</option>
                  <option>Globe</option>
                </select>
              </label>
            )}
          </div>
        </div>
      )}

      {block.type === "spacer" && (
        <div className="grid md:grid-cols-3 gap-2">
          <label className="label">Size<select className="input" value={block.size || "md"} onChange={(e) => onChange({ ...block, size: e.target.value })}><option>sm</option><option>md</option><option>lg</option></select></label>
        </div>
      )}

      <style jsx>{`
        .btn-small { @apply bg-white/15 hover:bg-white/25 px-2 py-1 rounded-lg text-xs transition; }
        .label { @apply flex flex-col text-white/90 text-sm; }
        .input { @apply bg-white text-gray-900 rounded-xl px-3 py-2 shadow-sm outline-none focus:ring-2 focus:ring-indigo-400; }
      `}</style>
    </div>
  );
}

function AddBlockBar({ onAdd }) {
  return (
    <div className="mt-8 p-3 bg-white/10 border border-white/20 rounded-xl text-white">
      <div className="font-semibold mb-2">Add block</div>
      <div className="flex flex-wrap gap-2">
        <button className="chip" onClick={() => onAdd({ type: "heading", text: "New heading", align: "inherit", size: "lg", weight: "semibold" })}>Heading</button>
        <button className="chip" onClick={() => onAdd({ type: "text", text: "Type **Markdown** here", align: "inherit" })}>Text (MD)</button>
        <button className="chip" onClick={() => onAdd({ type: "link", title: "New Link", url: "https://", subtitle: "", badge: "", variant: "inherit", icon: { mode: "none" } })}>Link</button>
        <button className="chip" onClick={() => onAdd({ type: "divider" })}>Divider</button>
        <button className="chip" onClick={() => onAdd({ type: "spacer", size: "md" })}>Spacer</button>
      </div>
      <style jsx>{`
        .chip { @apply bg-white/15 hover:bg-white/25 px-3 py-1.5 rounded-xl text-sm transition; }
      `}</style>
    </div>
  );
}

function SettingsModal({ data, onClose, onSave, importText, setImportText, onImport }) {
  const [layout, setLayout] = useState(data.layout);
  const [profile, setProfile] = useState({ ...data.profile, password: data.profile.password || DEFAULT_PASSWORD });

  return (
    <div className="fixed inset-0 z-30 flex items-center justify-center p-4">
      <div className="absolute inset-0 bg-black/50" onClick={onClose} />
      <div className="relative bg-white rounded-2xl w-full max-w-3xl p-4">
        <div className="flex items-center justify-between mb-3">
          <h3 className="text-lg font-semibold">Settings</h3>
          <button className="px-2 py-1 rounded-lg bg-gray-100" onClick={onClose}>Close</button>
        </div>
        <div className="grid md:grid-cols-2 gap-4">
          <section>
            <h4 className="font-semibold mb-2">Layout</h4>
            <div className="grid grid-cols-2 gap-2">
              <label className="label">Align<select className="input" value={layout.align} onChange={(e) => setLayout({ ...layout, align: e.target.value })}><option>center</option><option>left</option></select></label>
              <label className="label">Columns<select className="input" value={layout.columns} onChange={(e) => setLayout({ ...layout, columns: Number(e.target.value) })}><option value={1}>1</option><option value={2}>2</option></select></label>
              <label className="label">Width<select className="input" value={layout.width} onChange={(e) => setLayout({ ...layout, width: e.target.value })}><option>sm</option><option>md</option><option>lg</option></select></label>
            </div>
            <div className="mt-4">
              <button className="px-3 py-2 rounded-xl bg-indigo-600 text-white" onClick={() => onSave({ layout })}>Save Layout</button>
            </div>
          </section>

          <section>
            <h4 className="font-semibold mb-2">Profile & Security</h4>
            <div className="grid gap-2">
              <label className="label">Username<input className="input" value={profile.username} onChange={(e) => setProfile({ ...profile, username: e.target.value })} /></label>
              <label className="label">Password (edit mode)<input className="input" type="password" value={profile.password} onChange={(e) => setProfile({ ...profile, password: e.target.value })} /></label>
              <label className="label">Link Style<select className="input" value={profile.linkStyle} onChange={(e) => setProfile({ ...profile, linkStyle: e.target.value })}><option>solid</option><option>outline</option><option>ghost</option></select></label>
              <div>
                <button className="px-3 py-2 rounded-xl bg-indigo-600 text-white" onClick={() => onSave({ profile })}>Save Profile</button>
              </div>
            </div>
          </section>
        </div>

        <div className="mt-6 grid md:grid-cols-2 gap-4">
          <section>
            <h4 className="font-semibold mb-2">Import JSON</h4>
            <textarea className="w-full h-32 border rounded-xl p-2 text-sm" placeholder="Paste JSON here" value={importText} onChange={(e) => setImportText(e.target.value)} />
            <div className="mt-2 flex gap-2">
              <button className="px-3 py-2 rounded-xl bg-gray-200" onClick={() => setImportText("")}>Clear</button>
              <button className="px-3 py-2 rounded-xl bg-emerald-600 text-white" onClick={onImport}>Import</button>
            </div>
          </section>
          <section>
            <h4 className="font-semibold mb-2">Backup / Reset</h4>
            <div className="flex flex-wrap gap-2">
              <button className="px-3 py-2 rounded-xl bg-gray-200" onClick={() => { localStorage.removeItem(LS_KEY); location.reload(); }}>Reset to Default</button>
              <button className="px-3 py-2 rounded-xl bg-gray-200" onClick={() => navigator.clipboard.writeText(JSON.stringify(data, null, 2))}>Copy JSON</button>
            </div>
          </section>
        </div>
      </div>
    </div>
  );
}

/* ------------------------------ Utils ------------------------------ */
function linkButtonClass(variant) {
  const base = "block w-full rounded-2xl px-5 py-4 text-white shadow-lg transition transform hover:scale-[1.01] border";
  if (variant === "outline") return `${base} bg-white/10 border-white/40 backdrop-blur`;
  if (variant === "ghost") return `${base} bg-white/0 border-white/20 hover:bg-white/10`;
  return `${base} bg-white/20 border-white/20`;
}

function clsAlign(align) {
  if (align === "left") return "text-left";
  if (align === "center") return "text-center";
  return ""; // inherit from global
}

function optInherit(v) {
  return v === "inherit" ? undefined : v;
}

/* =====================
   📦 Self-Host with Docker
   ===================== */

/**
 * Project Structure (suggested)
 *
 * linktree-clone/
 * ├─ pages/
 * │  └─ index.jsx               // (this file) Next.js app page
 * ├─ public/
 * │  └─ favicon.ico             // optional
 * ├─ package.json
 * ├─ next.config.js
 * ├─ Dockerfile
 * ├─ docker-compose.yml
 * ├─ .dockerignore
 * └─ README.md                  // optional
 */

// next.config.js — enable standalone output for lean runtime image
export const nextConfig = {
  output: 'standalone',
};
export default nextConfig;

/* If using CommonJS instead (older Next.js):
module.exports = {
  output: 'standalone',
};
*/

// package.json — minimal scripts (if you used create-next-app you already have similar)
{
  "name": "linktree-clone",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start -p ${PORT:-3000}",
    "lint": "next lint"
  },
  "dependencies": {
    "lucide-react": "^0.460.0",
    "next": "^14.2.5",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-markdown": "^9.0.1"
  }
}

// .dockerignore — keep images small
node_modules
.next/cache
.git
.gitignore
Dockerfile
README.md
npm-debug.log

// Dockerfile — multi-stage build for Next.js standalone
# ---------- Builder ----------
FROM node:20-alpine AS builder
WORKDIR /app

# Install deps first (better layer caching)
COPY package*.json ./
RUN npm ci --no-audit --no-fund

# Copy source and build
COPY . .
RUN npm run build

# ---------- Runner ----------
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Copy the standalone output & public assets produced by Next.js
# .next/standalone contains the server code + node_modules needed at runtime
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

# Default port for Next.js
ENV PORT=3000
EXPOSE 3000

# Use non-root user when possible
RUN addgroup -S app && adduser -S app -G app
USER app

CMD ["node", "server.js"]

# Notes:
# - Next standalone creates a server.js in .next/standalone
# - If your Next version differs, the entry may be: server.js, server/index.js, or app/server.js

// docker-compose.yml — one service, maps host 8080 -> container 3000
version: "3.9"
services:
  linktree:
    build: .
    container_name: linktree
    ports:
      - "8080:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000"]
      interval: 30s
      timeout: 5s
      retries: 5

/* =====================
   🧭 Run Instructions
   =====================
   1) Build & start
      docker compose up -d --build

   2) Open in browser
      http://localhost:8080

   3) Edit mode password (default): admin
      - Change it in Settings → Profile & Security

   4) Update code and redeploy
      docker compose up -d --build

   5) Logs
      docker compose logs -f linktree

   6) Stop
      docker compose down

   7) Host on custom domain (reverse proxy example: Caddy)
      Example Caddyfile:
        your.domain.com {
          reverse_proxy linktree:3000
        }
      (Run Caddy as another service or on host; ensure Docker network is shared.)

   Security tips:
   - Behind a reverse proxy, enable HTTPS (Let’s Encrypt via Caddy/Traefik/Nginx).
   - Because data is localStorage-only, no server data persists; back up via Export JSON.
   - If later you add DB/auth, add volumes/env secrets and secure headers at proxy.
*/