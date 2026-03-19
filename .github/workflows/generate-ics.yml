// ============================================================
//  TURN Events — ICS Feed Generator
//  Fetches events from Airtable and writes events.ics
//  Run automatically by GitHub Actions on a schedule
// ============================================================

const fs   = require('fs');
const path = require('path');

const TOKEN  = process.env.AIRTABLE_TOKEN;
const BASE   = 'appyYHDYWswUnJ3d9';
const TABLE  = 'tblKQ1u2zYE4udojZ';
const MONTHS_AHEAD = 6;

if (!TOKEN) {
  console.error('ERROR: AIRTABLE_TOKEN environment variable is not set.');
  process.exit(1);
}

// ── Fetch all records from Airtable ──────────────────────────
async function fetchAll() {
  let records = [], offset;
  do {
    const url = new URL(`https://api.airtable.com/v0/${BASE}/${TABLE}`);
    if (offset) url.searchParams.set('offset', offset);
    const res  = await fetch(url, { headers: { Authorization: `Bearer ${TOKEN}` } });
    const data = await res.json();
    if (data.error) throw new Error(`Airtable error: ${data.error.message}`);
    records = records.concat(data.records);
    offset  = data.offset;
  } while (offset);
  return records;
}

// ── Date helpers ─────────────────────────────────────────────
function addDays(date, n) {
  const d = new Date(date); d.setDate(d.getDate() + n); return d;
}
function addMonths(date, n) {
  const d = new Date(date); d.setMonth(d.getMonth() + n); return d;
}
function toYMD(date) {
  return date.toISOString().slice(0, 10);
}

// ── ICS formatting ───────────────────────────────────────────
const pad = n => String(n).padStart(2, '0');

function fmtIcsLocal(date) {
  return `${date.getFullYear()}${pad(date.getMonth()+1)}${pad(date.getDate())}T${pad(date.getHours())}${pad(date.getMinutes())}00`;
}
function fmtIcsDate(date) {
  return `${date.getFullYear()}${pad(date.getMonth()+1)}${pad(date.getDate())}`;
}
function escIcs(s) {
  return String(s || '')
    .replace(/\\/g, '\\\\')
    .replace(/;/g,  '\\;')
    .replace(/,/g,  '\\,')
    .replace(/\n/g, '\\n');
}
// Fold long ICS lines at 75 chars (RFC 5545 requirement)
function fold(line) {
  if (line.length <= 75) return line;
  let out = '', i = 0;
  while (i < line.length) {
    if (i === 0) {
      out += line.slice(0, 75);
      i = 75;
    } else {
      out += '\r\n ' + line.slice(i, i + 74);
      i += 74;
    }
  }
  return out;
}

// ── Expand a single record into occurrences ──────────────────
function expandRecord(r) {
  const f = r.fields;
  if (!f['Event Name'] || !f['Start Date & Time']) return [];

  const name      = f['Event Name']         || '';
  const allDay    = f['All Day']            || false;
  const location  = f['Location']           || '';
  const desc      = f['Description']        || '';
  const rsvp      = f['RSVP / Event Link']  || '';
  const recurring = typeof f['Recurring'] === 'object'
    ? f['Recurring'].name : (f['Recurring'] || 'None');

  const startBase = new Date(f['Start Date & Time']);
  const endBase   = f['End Date & Time']
    ? new Date(f['End Date & Time'])
    : new Date(startBase.getTime() + 3600000);
  const duration  = endBase - startBase;

  const cancelled = (f['Cancelled Dates'] || '')
    .split('\n').map(d => d.trim()).filter(Boolean);

  const today  = new Date(); today.setHours(0, 0, 0, 0);
  const cutoff = addMonths(today, MONTHS_AHEAD);
  const recEndRaw = f['Recurring End Date'];
  const recEnd    = recEndRaw ? new Date(recEndRaw) : cutoff;
  const maxEnd    = recEnd < cutoff ? recEnd : cutoff;

  const occurrences = [];

  if (!recurring || recurring === 'None') {
    if (startBase >= today) {
      occurrences.push({
        uid: `${r.id}-single@turn-tigard`,
        name, allDay, location, desc, rsvp,
        start: startBase,
        end:   new Date(startBase.getTime() + duration),
      });
    }
  } else {
    let cursor = new Date(startBase);
    let count  = 0;
    while (cursor <= maxEnd && count < 500) { // safety cap
      if (cursor >= today && !cancelled.includes(toYMD(cursor))) {
        occurrences.push({
          uid: `${r.id}-${toYMD(cursor)}@turn-tigard`,
          name, allDay, location, desc, rsvp,
          start: new Date(cursor),
          end:   new Date(cursor.getTime() + duration),
        });
      }
      if (recurring === 'Weekly')         cursor = addDays(cursor, 7);
      else if (recurring === 'Bi-weekly') cursor = addDays(cursor, 14);
      else if (recurring === 'Monthly')   cursor = addMonths(cursor, 1);
      else break;
      count++;
    }
  }

  return occurrences;
}

// ── Build VEVENT block ────────────────────────────────────────
function buildVEvent(ev) {
  const lines = ['BEGIN:VEVENT'];

  lines.push(`UID:${ev.uid}`);
  lines.push(`DTSTAMP:${fmtIcsLocal(new Date())}Z`);

  if (ev.allDay) {
    lines.push(`DTSTART;VALUE=DATE:${fmtIcsDate(ev.start)}`);
    lines.push(`DTEND;VALUE=DATE:${fmtIcsDate(addDays(ev.end, 1))}`);
  } else {
    lines.push(`DTSTART;TZID=America/Los_Angeles:${fmtIcsLocal(ev.start)}`);
    lines.push(`DTEND;TZID=America/Los_Angeles:${fmtIcsLocal(ev.end)}`);
  }

  lines.push(fold(`SUMMARY:${escIcs(ev.name)}`));
  if (ev.desc)     lines.push(fold(`DESCRIPTION:${escIcs(ev.desc)}`));
  if (ev.location) lines.push(fold(`LOCATION:${escIcs(ev.location)}`));
  if (ev.rsvp)     lines.push(fold(`URL:${ev.rsvp}`));

  lines.push('END:VEVENT');
  return lines.join('\r\n');
}

// ── Main ─────────────────────────────────────────────────────
async function main() {
  console.log('Fetching events from Airtable…');
  const records = await fetchAll();
  console.log(`  ${records.length} records fetched`);

  const events = records.flatMap(expandRecord);
  events.sort((a, b) => a.start - b.start);
  console.log(`  ${events.length} occurrences generated`);

  const ics = [
    'BEGIN:VCALENDAR',
    'VERSION:2.0',
    'PRODID:-//Tigard United Resistance Network//Events//EN',
    'X-WR-CALNAME:TURN Events',
    'X-WR-CALDESC:Events from Tigard United Resistance Network – Tigard OR',
    'X-WR-TIMEZONE:America/Los_Angeles',
    'CALSCALE:GREGORIAN',
    'METHOD:PUBLISH',
    ...events.map(buildVEvent),
    'END:VCALENDAR',
  ].join('\r\n');

  const outPath = path.join(__dirname, 'events.ics');
  fs.writeFileSync(outPath, ics, 'utf8');
  console.log(`  Written to ${outPath}`);
}

main().catch(err => { console.error(err); process.exit(1); });
