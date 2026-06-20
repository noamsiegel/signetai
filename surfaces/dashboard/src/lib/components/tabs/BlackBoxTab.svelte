<script lang="ts">
import {
	type BlackBoxEvent,
	type BlackBoxRef,
	type BlackBoxSession,
	type BlackBoxSessionSummary,
	getBlackBoxSession,
	getBlackBoxSessions,
} from "$lib/api";

interface Props {
	agentId: string;
}

type TimelinePhase = "start" | "recall" | "write" | "graph" | "other";

interface TimelineGroup {
	phase: TimelinePhase;
	label: string;
	description: string;
	items: Array<{ event: BlackBoxEvent; index: number }>;
}

interface GraphNode {
	id: string;
	ref: BlackBoxRef | null;
	label: string;
	kind: string;
	x: number;
	y: number;
	trust: readonly string[];
}

const { agentId }: Props = $props();

let sessions = $state<BlackBoxSessionSummary[]>([]);
let selectedSessionKey = $state("");
let manualSessionKey = $state("");
let replay = $state<BlackBoxSession | null>(null);
let selectedIndex = $state(0);
let loadingSessions = $state(false);
let loadingReplay = $state(false);
let error = $state<string | null>(null);
let demoMode = $state(false);
let explainOpen = $state(true);
let loadedAgentId = $state("");
let sessionsRequestId = 0;
let replayRequestId = 0;

function createDemoReplay(currentAgentId: string): BlackBoxSession {
	return {
		sessionKey: "demo:black-box-flight-recorder",
		agentId: currentAgentId,
		generatedAt: new Date().toISOString(),
		eventCount: 6,
		events: [
			{
				id: "demo:start",
				kind: "recall.requested",
				at: "2026-06-20T10:00:00.000Z",
				title: "Session started",
				detail: "The agent opened a scoped working session for dashboard investigation.",
				refs: [],
				payload: { route: "session.start" },
			},
			{
				id: "demo:recall-query",
				kind: "recall.requested",
				at: "2026-06-20T10:00:18.000Z",
				title: "Recall requested",
				detail: "dashboard graph provenance UX",
				refs: [],
				payload: { route: "/api/memory/recall" },
			},
			{
				id: "demo:recall-results",
				kind: "recall.result",
				at: "2026-06-20T10:00:19.000Z",
				title: "3 context references returned",
				detail: "The recall pass returned source-backed design notes and one memory preference.",
				refs: [
					{
						kind: "source",
						id: "docs/specs/dashboard-overhaul-brief.md#north-star",
						label: "Dashboard north star: graph-first workspace",
						score: 0.91,
						sourcePath: "docs/specs/dashboard-overhaul-brief.md",
					},
					{
						kind: "source_artifact",
						id: "surfaces/dashboard/DESIGN.md#tokens",
						label: "Signet Dashboard design contract",
						score: 0.86,
						sourcePath: "surfaces/dashboard/DESIGN.md",
					},
					{
						kind: "memory",
						id: "mem:local-first-provenance",
						label: "Prefer source-backed claims with provenance",
						score: 0.78,
					},
				],
			},
			{
				id: "demo:context-injected",
				kind: "context.recalled",
				at: "2026-06-20T10:00:23.000Z",
				title: "Context entered prompt",
				detail: "The graph-first dashboard brief became active context for the answer.",
				refs: [
					{
						kind: "source",
						id: "docs/specs/dashboard-overhaul-brief.md#north-star",
						label: "Dashboard north star: graph-first workspace",
						score: 0.91,
						sourcePath: "docs/specs/dashboard-overhaul-brief.md",
					},
				],
			},
			{
				id: "demo:assertion",
				kind: "assertion.created",
				at: "2026-06-20T10:01:02.000Z",
				title: "Assertion observed",
				detail: "Black Box should explain influence evidence, not claim model causality.",
				refs: [
					{
						kind: "assertion",
						id: "assert:black-box-causality-caveat",
						label: "Influence evidence is not guaranteed causality",
						status: "active",
						sourcePath: "session:demo:black-box-flight-recorder",
					},
				],
			},
			{
				id: "demo:summary",
				kind: "artifact.written",
				at: "2026-06-20T10:02:30.000Z",
				title: "Summary artifact written",
				detail: "The session wrote a durable local artifact that can be replayed later.",
				refs: [
					{
						kind: "source_artifact",
						id: "memory/sessions/demo-black-box.md",
						label: "Session summary artifact",
						sourcePath: "memory/sessions/demo-black-box.md",
					},
				],
			},
		],
		frame: {
			at: "2026-06-20T10:02:30.000Z",
			activeRefCount: 5,
			activeRefs: [],
			likelyInfluences: [],
			warnings: [],
		},
	};
}

const selectedEvent = $derived.by(() => replay?.events[selectedIndex] ?? null);
const activeRefsAtSelection = $derived.by(() => {
	if (!replay) return [] as BlackBoxRef[];
	const refs = new Map<string, BlackBoxRef>();
	for (const event of replay.events.slice(0, selectedIndex + 1)) {
		for (const ref of event.refs) refs.set(`${ref.kind}:${ref.id}`, ref);
	}
	return [...refs.values()];
});
const visibleRefs = $derived.by(() => activeRefsAtSelection.slice(0, 20));
const timelineGroups = $derived.by(() => groupTimelineEvents(replay?.events ?? []));
const likelyInfluencesAtSelection = $derived.by(() => {
	if (!replay) return [];
	return replay.events
		.slice(0, selectedIndex + 1)
		.filter((event) => event.kind === "recall.result" || event.kind === "context.recalled")
		.slice(-5)
		.reverse()
		.map((event) => ({
			at: event.at,
			reason:
				event.kind === "recall.result"
					? "Returned by explicit recall before this moment"
					: "Already injected in this context epoch",
			refs: event.refs,
		}));
});
const graphNodes = $derived.by(() => buildGraphNodes(visibleRefs, replay?.sessionKey ?? selectedSessionKey));
const graphRefNodes = $derived.by(() => graphNodes.filter((node) => node.id !== "session"));
const eventCountLabel = $derived.by(() => `${replay?.eventCount ?? 0} EVENTS`);
const explanationLines = $derived.by(() =>
	buildExplanationLines(selectedEvent, likelyInfluencesAtSelection, activeRefsAtSelection),
);

function formatTime(value: string): string {
	const date = new Date(value);
	if (Number.isNaN(date.getTime())) return value;
	return date.toLocaleString(undefined, {
		month: "short",
		day: "2-digit",
		hour: "2-digit",
		minute: "2-digit",
		second: "2-digit",
	});
}

function phaseForEvent(event: BlackBoxEvent, index: number): TimelinePhase {
	if (index === 0 && event.kind === "recall.requested") return "start";
	if (event.kind === "recall.requested" || event.kind === "recall.result" || event.kind === "context.recalled") {
		return "recall";
	}
	if (event.kind === "artifact.written") return "write";
	if (event.kind === "assertion.created") return "graph";
	return "other";
}

function phaseLabel(phase: TimelinePhase): string {
	switch (phase) {
		case "start":
			return "Start";
		case "recall":
			return "Recall";
		case "write":
			return "Write";
		case "graph":
			return "Graph";
		case "other":
			return "Other";
	}
}

function phaseDescription(phase: TimelinePhase): string {
	switch (phase) {
		case "start":
			return "session and scope context";
		case "recall":
			return "context retrieved or injected";
		case "write":
			return "durable local artifacts";
		case "graph":
			return "claims and provenance";
		case "other":
			return "supporting events";
	}
}

function groupTimelineEvents(events: readonly BlackBoxEvent[]): TimelineGroup[] {
	const groups: TimelineGroup[] = [];
	for (const [index, event] of events.entries()) {
		const phase = phaseForEvent(event, index);
		let group = groups.at(-1);
		if (!group || group.phase !== phase) {
			group = { phase, label: phaseLabel(phase), description: phaseDescription(phase), items: [] };
			groups.push(group);
		}
		group.items.push({ event, index });
	}
	return groups;
}

function eventClass(event: BlackBoxEvent): string {
	return `timeline-event event-${event.kind.replace(".", "-")}`;
}

function refClass(ref: BlackBoxRef): string {
	return `ref-chip ref-${ref.kind.replace("_", "-")}`;
}

function nodeClass(node: GraphNode): string {
	return `graph-node graph-node-${node.kind.replace("_", "-")}`;
}

function scoreLabel(score: number | undefined): string {
	if (typeof score !== "number") return "";
	return score.toFixed(2);
}

function graphNodeStyle(node: GraphNode): string {
	return `left:${node.x}%;top:${node.y}%`;
}

function refTrust(ref: BlackBoxRef): readonly string[] {
	const badges: string[] = [];
	if (ref.status === "superseded" || ref.status === "archived") badges.push(ref.status);
	if (ref.kind === "source" || ref.kind === "source_artifact" || ref.sourcePath) badges.push("source-backed");
	if (ref.kind === "assertion") badges.push("claim");
	if (typeof ref.score === "number") badges.push(`score ${scoreLabel(ref.score)}`);
	if (badges.length === 0) badges.push("memory-only");
	if (!ref.sourcePath && ref.kind === "memory") badges.push("no source path");
	return badges;
}

function eventTrust(event: BlackBoxEvent): readonly string[] {
	const badges = new Set<string>();
	for (const ref of event.refs) {
		for (const badge of refTrust(ref)) badges.add(badge);
	}
	if (event.refs.length === 0) badges.add("control event");
	return [...badges].slice(0, 4);
}

function buildGraphNodes(refs: readonly BlackBoxRef[], sessionKey: string): GraphNode[] {
	const nodes: GraphNode[] = [
		{
			id: "session",
			ref: null,
			label: sessionKey || "session",
			kind: "session",
			x: 50,
			y: 50,
			trust: ["current frame"],
		},
	];
	const count = Math.max(refs.length, 1);
	for (const [index, ref] of refs.entries()) {
		const angle = -Math.PI / 2 + (Math.PI * 2 * index) / count;
		const ring = ref.kind === "memory" ? 27 : ref.kind === "assertion" ? 36 : 32;
		nodes.push({
			id: `${ref.kind}:${ref.id}`,
			ref,
			label: ref.label ?? ref.id,
			kind: ref.kind,
			x: 50 + Math.cos(angle) * ring,
			y: 50 + Math.sin(angle) * ring * 0.72,
			trust: refTrust(ref),
		});
	}
	return nodes;
}

function buildExplanationLines(
	event: BlackBoxEvent | null,
	influences: readonly { readonly reason: string; readonly refs: readonly BlackBoxRef[] }[],
	activeRefs: readonly BlackBoxRef[],
): string[] {
	if (!event) return ["Select a timeline event to explain this frame."];
	const lines = [`Selected event: ${event.title}.`];
	if (event.refs.length > 0) {
		lines.push(
			`${event.refs.length} reference${event.refs.length === 1 ? "" : "s"} changed or became inspectable at this point.`,
		);
	} else {
		lines.push("This is a control event; no context reference was attached directly.");
	}
	if (influences.length > 0) {
		const top = influences[0];
		const labels = top.refs.map((ref) => ref.label ?? ref.id).slice(0, 2);
		lines.push(
			`Most recent influence evidence: ${top.reason.toLowerCase()}${labels.length ? ` (${labels.join("; ")})` : ""}.`,
		);
	}
	const sourceBacked = activeRefs.filter(
		(ref) => ref.sourcePath || ref.kind === "source" || ref.kind === "source_artifact",
	).length;
	lines.push(`${sourceBacked}/${activeRefs.length} active references are source-backed in this frame.`);
	return lines;
}

function loadDemo(): void {
	sessionsRequestId += 1;
	replayRequestId += 1;
	loadingSessions = false;
	loadingReplay = false;
	demoMode = true;
	error = null;
	const demoReplay = createDemoReplay(agentId);
	replay = demoReplay;
	selectedSessionKey = demoReplay.sessionKey;
	manualSessionKey = demoReplay.sessionKey;
	selectedIndex = Math.max(0, demoReplay.events.length - 1);
}

async function loadSessions(currentAgentId = agentId): Promise<void> {
	const requestId = ++sessionsRequestId;
	const replayRequestIdAtStart = replayRequestId;
	loadingSessions = true;
	error = null;
	try {
		const data = await getBlackBoxSessions(currentAgentId, 50);
		if (requestId !== sessionsRequestId || currentAgentId !== agentId) return;
		sessions = data.sessions;
		if (
			!demoMode &&
			replayRequestId === replayRequestIdAtStart &&
			!replay &&
			!selectedSessionKey &&
			!manualSessionKey &&
			sessions[0]
		) {
			selectedSessionKey = sessions[0].sessionKey;
			manualSessionKey = sessions[0].sessionKey;
			await loadReplay(sessions[0].sessionKey, currentAgentId);
		}
	} catch (err) {
		if (requestId === sessionsRequestId) error = err instanceof Error ? err.message : String(err);
	} finally {
		if (requestId === sessionsRequestId) loadingSessions = false;
	}
}

async function loadReplay(sessionKey = selectedSessionKey, currentAgentId = agentId): Promise<void> {
	const key = sessionKey.trim();
	if (!key) return;
	const requestId = ++replayRequestId;
	loadingReplay = true;
	error = null;
	demoMode = false;
	try {
		const nextReplay = await getBlackBoxSession(key, { agentId: currentAgentId, limit: 500 });
		if (requestId !== replayRequestId || currentAgentId !== agentId) return;
		replay = nextReplay;
		selectedSessionKey = key;
		manualSessionKey = key;
		selectedIndex = Math.max(0, replay.events.length - 1);
	} catch (err) {
		if (requestId !== replayRequestId) return;
		replay = null;
		error = err instanceof Error ? err.message : String(err);
	} finally {
		if (requestId === replayRequestId) loadingReplay = false;
	}
}

function handleSessionSelect(event: Event): void {
	const target = event.currentTarget;
	if (!(target instanceof HTMLSelectElement)) return;
	void loadReplay(target.value);
}

function handleManualSubmit(): void {
	void loadReplay(manualSessionKey);
}

function toggleExplain(): void {
	explainOpen = !explainOpen;
}

$effect(() => {
	const currentAgentId = agentId;
	if (!currentAgentId || currentAgentId === loadedAgentId) return;
	loadedAgentId = currentAgentId;
	if (!demoMode) {
		replay = null;
		selectedSessionKey = "";
		manualSessionKey = "";
		selectedIndex = 0;
	}
	void loadSessions(currentAgentId);
});
</script>

<div class="blackbox-tab">
	<header class="blackbox-header">
		<div>
			<div class="eyebrow">BLACK BOX MODE</div>
			<h2>Debug the ghost in the agent</h2>
			<p>Replay what this session could see: recall hits, context injections, session artifacts, and provenance-bearing assertions.</p>
		</div>
		<div class="header-meta">
			<span>{demoMode ? "demo" : agentId}</span>
			<span>{eventCountLabel}</span>
		</div>
	</header>

	<section class="controls" aria-label="Black Box session controls">
		<label>
			<span>Recorded sessions</span>
			<select disabled={loadingSessions || sessions.length === 0} onchange={handleSessionSelect} bind:value={selectedSessionKey}>
				{#if sessions.length === 0}
					<option value="">No sessions found</option>
				{:else}
					{#each sessions as session}
						<option value={session.sessionKey}>
							{session.sessionKey} · {formatTime(session.lastAt)} · {session.recallEvents + session.artifactEvents} events
						</option>
					{/each}
				{/if}
			</select>
		</label>
		<label class="manual-key">
			<span>Session key</span>
			<input bind:value={manualSessionKey} placeholder="Paste a session key" onkeydown={(event) => event.key === "Enter" && handleManualSubmit()} />
		</label>
		<div class="control-actions">
			<button type="button" onclick={handleManualSubmit} disabled={loadingReplay || !manualSessionKey.trim()}>Load</button>
			<button type="button" class="ghost-button" onclick={loadDemo}>Demo</button>
		</div>
	</section>

	{#if error}
		<div class="error-panel">{error}</div>
	{/if}

	{#if loadingReplay && !replay}
		<div class="empty-panel">Loading flight recorder…</div>
	{:else if replay && replay.events.length > 0}
		<div class="replay-grid">
			<section class="timeline-panel" aria-label="Session timeline">
				<div class="panel-title">
					<span>TIMELINE</span>
					<small>{selectedEvent ? formatTime(selectedEvent.at) : formatTime(replay.frame.at)}</small>
				</div>
				<input
					class="scrubber"
					type="range"
					min="0"
					max={Math.max(0, replay.events.length - 1)}
					bind:value={selectedIndex}
				/>
				<div class="event-list">
					{#each timelineGroups as group}
						<div class="phase-row">
							<span>{group.label}</span>
							<small>{group.description}</small>
						</div>
						{#each group.items as item}
							<button
								type="button"
								class:event-selected={item.index === selectedIndex}
								class={eventClass(item.event)}
								onclick={() => (selectedIndex = item.index)}
							>
								<span class="event-dot" aria-hidden="true"></span>
								<span class="event-main">
									<strong>{item.event.title}</strong>
									<small>{formatTime(item.event.at)}</small>
								</span>
								<span class="event-ref-count">{item.event.refs.length}</span>
							</button>
						{/each}
					{/each}
				</div>
			</section>

			<section class="graph-panel" aria-label="Context graph replay">
				<div class="panel-title">
					<span>FRAME GRAPH</span>
					<small>{activeRefsAtSelection.length} active refs</small>
				</div>
				<div class="context-field">
					<svg class="edge-layer" viewBox="0 0 100 100" aria-hidden="true">
						{#each graphRefNodes as node}
							<line x1="50" y1="50" x2={node.x} y2={node.y} />
						{/each}
					</svg>
					{#each graphNodes as node}
						<div class={nodeClass(node)} style={graphNodeStyle(node)} title={node.label}>
							<span>{node.kind}</span>
							<strong>{node.label}</strong>
							<div class="node-badges">
								{#each node.trust.slice(0, 2) as badge}
									<small>{badge}</small>
								{/each}
							</div>
						</div>
					{/each}
					{#if visibleRefs.length === 0}
						<div class="field-empty">No context refs before this frame.</div>
					{/if}
				</div>
			</section>

			<section class="inspector-panel" aria-label="Black Box inspector">
				<div class="panel-title">
					<span>INSPECTOR</span>
					<button type="button" class="mini-button" onclick={toggleExplain}>
						{explainOpen ? "Hide why" : "Why this?"}
					</button>
				</div>
				{#if selectedEvent}
					<div class="event-kind">{selectedEvent.kind}</div>
					<h3>{selectedEvent.title}</h3>
					<p>{selectedEvent.detail}</p>
					<div class="trust-strip">
						{#each eventTrust(selectedEvent) as badge}
							<span>{badge}</span>
						{/each}
					</div>
					{#if explainOpen}
						<div class="why-card">
							<strong>Why this moment?</strong>
							{#each explanationLines as line}
								<span>{line}</span>
							{/each}
						</div>
					{/if}
					<div class="ref-list">
						{#each selectedEvent.refs as ref}
							<div class="ref-row">
								<span>{ref.kind}</span>
								<strong>{ref.label ?? ref.id}</strong>
								<div class="row-badges">
									{#each refTrust(ref).slice(0, 3) as badge}<small>{badge}</small>{/each}
								</div>
							</div>
						{:else}
							<div class="field-empty">No direct refs on this event.</div>
						{/each}
					</div>
				{:else}
					<p>Select a timeline event.</p>
				{/if}
			</section>

			<section class="influence-panel" aria-label="Likely influences">
				<div class="panel-title"><span>LIKELY INFLUENCES</span></div>
				{#each likelyInfluencesAtSelection as influence}
					<div class="influence-card">
						<small>{formatTime(influence.at)}</small>
						<strong>{influence.reason}</strong>
						<span>{influence.refs.map((ref) => ref.label ?? ref.id).slice(0, 3).join(" · ")}</span>
					</div>
				{:else}
					<div class="field-empty">No recall evidence before this frame.</div>
				{/each}
			</section>
		</div>
	{:else}
		<div class="empty-panel empty-demo">
			<strong>No flight-recorder data yet.</strong>
			<span>Run a session with recall telemetry, paste a session key, or open the guided demo to see how Black Box explains a session.</span>
			<button type="button" onclick={loadDemo}>Open demo replay</button>
		</div>
	{/if}
</div>

<style>
	.blackbox-tab {
		display: flex;
		flex-direction: column;
		flex: 1;
		min-height: 0;
		overflow: hidden;
		background:
			linear-gradient(90deg, rgba(246, 246, 244, 0.035) 1px, transparent 1px),
			linear-gradient(180deg, rgba(246, 246, 244, 0.025) 1px, transparent 1px),
			var(--sig-bg);
		background-size: 48px 48px;
	}

	.blackbox-header {
		display: flex;
		justify-content: space-between;
		gap: var(--space-md);
		padding: var(--space-md);
		border-bottom: 1px solid var(--sig-border);
	}

	.eyebrow,
	.panel-title,
	.header-meta,
	.controls span,
	.event-kind,
	.phase-row {
		font-family: var(--font-display);
		font-size: 10px;
		letter-spacing: 0.1em;
		text-transform: uppercase;
		color: var(--sig-text-muted);
	}

	h2 {
		margin: 4px 0;
		font-family: var(--font-display);
		font-size: 18px;
		letter-spacing: 0.04em;
		color: var(--sig-text-bright);
	}

	p {
		max-width: 760px;
		margin: 0;
		color: var(--sig-text-muted);
		font-size: 13px;
	}

	.header-meta,
	.control-actions,
	.trust-strip,
	.row-badges,
	.node-badges {
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.header-meta {
		align-items: flex-start;
		white-space: nowrap;
	}

	.header-meta span,
	.event-ref-count,
	.trust-strip span,
	.row-badges small,
	.node-badges small {
		border: 1px solid var(--sig-border);
		background: var(--sig-surface);
		padding: 4px 7px;
	}

	.controls {
		display: grid;
		grid-template-columns: minmax(260px, 1fr) minmax(220px, 0.8fr) auto;
		gap: 8px;
		padding: var(--space-sm) var(--space-md);
		border-bottom: 1px solid var(--sig-border);
		background: rgba(17, 21, 32, 0.82);
	}

	label {
		display: flex;
		flex-direction: column;
		gap: 4px;
	}

	select,
	input,
	button {
		border: 1px solid var(--sig-border);
		background: var(--sig-surface);
		color: var(--sig-text);
		padding: 8px 10px;
		font: inherit;
	}

	button {
		align-self: end;
		cursor: pointer;
		text-transform: uppercase;
		letter-spacing: 0.08em;
		font-size: 11px;
	}

	button:hover:not(:disabled) {
		border-color: var(--sig-highlight);
		color: var(--sig-highlight);
	}

	button:disabled,
	select:disabled {
		opacity: 0.55;
		cursor: not-allowed;
	}

	.ghost-button,
	.mini-button {
		background: transparent;
	}

	.mini-button {
		padding: 3px 6px;
		font-size: 9px;
	}

	.error-panel,
	.empty-panel {
		margin: var(--space-md);
		padding: var(--space-md);
		border: 1px solid var(--sig-border);
		background: var(--sig-surface);
		color: var(--sig-text-muted);
	}

	.error-panel {
		border-color: rgba(255, 90, 31, 0.55);
		color: var(--sig-danger, #ff5a1f);
	}

	.empty-panel {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.empty-demo {
		max-width: 620px;
	}

	.empty-demo button {
		align-self: flex-start;
		margin-top: 6px;
	}

	.replay-grid {
		display: grid;
		grid-template-columns: minmax(280px, 0.82fr) minmax(440px, 1.38fr) minmax(320px, 0.95fr);
		grid-template-rows: minmax(0, 1fr) minmax(180px, 0.44fr);
		gap: var(--space-sm);
		padding: var(--space-md);
		min-height: 0;
		flex: 1;
	}

	.timeline-panel,
	.graph-panel,
	.inspector-panel,
	.influence-panel {
		min-height: 0;
		border: 1px solid var(--sig-border);
		background: rgba(11, 14, 23, 0.88);
	}

	.timeline-panel,
	.inspector-panel,
	.influence-panel {
		display: flex;
		flex-direction: column;
	}

	.timeline-panel {
		grid-row: 1 / span 2;
	}

	.graph-panel {
		grid-column: 2;
		grid-row: 1 / span 2;
		display: flex;
		flex-direction: column;
	}

	.panel-title {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 8px;
		padding: 10px;
		border-bottom: 1px solid var(--sig-border);
	}

	.scrubber {
		width: calc(100% - 20px);
		margin: 10px;
		padding: 0;
	}

	.event-list,
	.ref-list,
	.influence-panel {
		overflow: auto;
	}

	.phase-row {
		display: flex;
		justify-content: space-between;
		gap: 8px;
		padding: 8px 10px;
		border-top: 1px solid rgba(246, 246, 244, 0.08);
		border-bottom: 1px solid rgba(246, 246, 244, 0.08);
		background: rgba(246, 246, 244, 0.025);
	}

	.phase-row span {
		color: var(--sig-text-bright);
	}

	.phase-row small {
		font-size: 9px;
	}

	.timeline-event {
		display: grid;
		grid-template-columns: 12px 1fr auto;
		align-items: center;
		gap: 8px;
		width: 100%;
		padding: 9px 10px;
		border: 0;
		border-bottom: 1px solid var(--sig-border);
		background: transparent;
		text-align: left;
	}

	.timeline-event.event-selected {
		background: rgba(10, 57, 255, 0.14);
		color: var(--sig-text-bright);
	}

	.event-dot {
		width: 7px;
		height: 7px;
		border: 1px solid var(--sig-text-muted);
		background: transparent;
		transform: rotate(45deg);
	}

	.event-recall-result .event-dot,
	.event-context-recalled .event-dot {
		border-color: var(--sig-highlight);
		background: rgba(10, 57, 255, 0.55);
	}

	.event-artifact-written .event-dot,
	.event-assertion-created .event-dot {
		border-color: #ff5a1f;
	}

	.event-main {
		display: flex;
		flex-direction: column;
		gap: 3px;
		min-width: 0;
	}

	.event-main strong,
	.ref-row strong,
	.graph-node strong,
	.influence-card strong {
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	.event-main small,
	.ref-row small,
	.graph-node small,
	.influence-card small {
		color: var(--sig-text-muted);
		font-size: 10px;
	}

	.context-field {
		position: relative;
		flex: 1;
		min-height: 0;
		overflow: hidden;
		background:
			radial-gradient(circle at 50% 50%, rgba(10, 57, 255, 0.17), transparent 22%),
			radial-gradient(circle at 24% 20%, rgba(255, 90, 31, 0.1), transparent 20%);
	}

	.edge-layer {
		position: absolute;
		inset: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	.edge-layer line {
		stroke: rgba(246, 246, 244, 0.17);
		stroke-width: 0.25;
	}

	.graph-node {
		position: absolute;
		display: flex;
		flex-direction: column;
		max-width: 190px;
		min-width: 112px;
		gap: 3px;
		padding: 7px 8px;
		border: 1px solid rgba(246, 246, 244, 0.16);
		background: rgba(17, 21, 32, 0.94);
		box-shadow: 0 0 0 1px rgba(10, 57, 255, 0.08);
		transform: translate(-50%, -50%);
	}

	.graph-node span,
	.ref-row span {
		font-size: 9px;
		text-transform: uppercase;
		letter-spacing: 0.08em;
		color: var(--sig-text-muted);
	}

	.graph-node-session {
		min-width: 170px;
		border-color: rgba(10, 57, 255, 0.72);
		background: rgba(10, 57, 255, 0.18);
	}

	.graph-node-memory {
		border-color: rgba(10, 57, 255, 0.5);
	}

	.graph-node-source,
	.graph-node-source-artifact {
		border-color: rgba(103, 209, 143, 0.5);
	}

	.graph-node-assertion {
		border-color: rgba(255, 90, 31, 0.5);
	}

	.node-badges,
	.row-badges {
		flex-wrap: wrap;
	}

	.node-badges small,
	.row-badges small,
	.trust-strip span {
		padding: 2px 5px;
		font-size: 9px;
	}

	.field-empty {
		padding: var(--space-md);
		color: var(--sig-text-muted);
		font-size: 12px;
	}

	.inspector-panel {
		padding-bottom: 10px;
	}

	.inspector-panel h3,
	.inspector-panel p,
	.event-kind,
	.trust-strip,
	.why-card {
		margin: 10px 12px 0;
	}

	.inspector-panel h3 {
		color: var(--sig-text-bright);
		font-size: 16px;
	}

	.trust-strip {
		flex-wrap: wrap;
	}

	.why-card {
		display: flex;
		flex-direction: column;
		gap: 6px;
		padding: 10px;
		border: 1px solid rgba(10, 57, 255, 0.34);
		background: rgba(10, 57, 255, 0.08);
		color: var(--sig-text-muted);
		font-size: 12px;
	}

	.why-card strong {
		color: var(--sig-text-bright);
	}

	.ref-row {
		display: grid;
		grid-template-columns: 78px minmax(0, 1fr);
		gap: 8px;
		align-items: start;
		padding: 8px 12px;
		border-top: 1px solid var(--sig-border);
	}

	.row-badges {
		grid-column: 2;
	}

	.influence-card {
		display: flex;
		flex-direction: column;
		gap: 4px;
		padding: 10px;
		border-bottom: 1px solid var(--sig-border);
	}

	.influence-card span {
		color: var(--sig-text-muted);
		font-size: 12px;
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	@media (max-width: 1100px) {
		.controls,
		.replay-grid {
			grid-template-columns: 1fr;
		}

		.replay-grid {
			grid-template-rows: auto;
			overflow: auto;
		}

		.timeline-panel,
		.graph-panel {
			grid-column: auto;
			grid-row: auto;
			min-height: 320px;
		}
	}
</style>
