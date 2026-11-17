<%*
const activeFile = app.workspace.getActiveFile();
if (!activeFile) return;

const cache = app.metadataCache.getFileCache(activeFile);

const tags = cache?.tags?.map(t => t.tag.toLowerCase().replace("#", "")) || [];
const fmTagsData = cache?.frontmatter?.tags || cache?.frontmatter?.tag;
let allTags = [...tags];

if (fmTagsData) {
    if (Array.isArray(fmTagsData)) {
        allTags.push(...fmTagsData.map(t => String(t).toLowerCase().replace("#", "")));
    } else if (typeof fmTagsData === "string") {
        allTags.push(...fmTagsData.split(/,\s*/).map(t => t.toLowerCase().replace("#", "")));
    }
}

const VALID_TYPES = ["sourcenote", "jotdown", "concept", "personal", "evergreen", "review", "idea"];
const activeTypeTag = VALID_TYPES.find(type => allTags.includes(type));
if (!activeTypeTag) return;
const activeType = activeTypeTag === "sourcenote" ? "source" : activeTypeTag;

let text = await app.vault.read(activeFile);
if (text.startsWith("---")) {
    const endFrontmatter = text.indexOf("---", 3);
    if (endFrontmatter !== -1) text = text.substring(endFrontmatter + 3);
}

let rawLines = text.split(/\r?\n/).map(l => l.trim());
let cleanLines = [];

if (activeType === "source") {
    let firstHeaderIndex = rawLines.findIndex(l => l.startsWith("#"));
    let secondHeaderIndex = -1;
    if (firstHeaderIndex !== -1) {
        secondHeaderIndex = rawLines.findIndex((l, idx) => idx > firstHeaderIndex && l.startsWith("#"));
    }
    
    let linesBeforeIntro = [];
    let linesAfterIntro = [];
    let introSectionLines = [];
    
    if (firstHeaderIndex !== -1 && secondHeaderIndex !== -1) {
        linesBeforeIntro = rawLines.slice(0, firstHeaderIndex);
        introSectionLines = rawLines.slice(firstHeaderIndex + 1, secondHeaderIndex);
        linesAfterIntro = rawLines.slice(secondHeaderIndex + 1);
    } else if (firstHeaderIndex !== -1) {
        linesBeforeIntro = rawLines.slice(0, firstHeaderIndex);
        introSectionLines = rawLines.slice(firstHeaderIndex + 1);
    } else {
        linesAfterIntro = rawLines;
    }
    
    let introTextBlocks = introSectionLines.join("\n").split(/\n\s*\n/).map(p => p.trim()).filter(p => p.length > 0);
    if (introTextBlocks.length > 0) {
        introTextBlocks.shift();
    }
    
    cleanLines = [
        ...linesBeforeIntro,
        ...introTextBlocks,
        ...linesAfterIntro
    ];
} else {
    cleanLines = [...rawLines];
}

cleanLines = cleanLines.filter(l => !l.startsWith("#"));
const finalizedText = cleanLines.join(" ");
const wordCount = finalizedText.split(/\s+/).filter(t => t.length > 0).length;

let totalOutgoingLinks = cache?.links ? cache.links.length : 0;
let validOutgoingLinks = 0;

if (cache && cache.links) {
    for (const link of cache.links) {
        const destPath = app.metadataCache.getFirstLinkpathDest(link.link, activeFile.path)?.path;
        if (!destPath) continue;
        const targetFile = app.vault.getAbstractFileByPath(destPath);
        if (!targetFile) continue;
        
        const targetCache = app.metadataCache.getFileCache(targetFile);
        const targetTags = targetCache?.tags?.map(t => t.tag.toLowerCase().replace("#", "")) || [];
        const targetFmData = targetCache?.frontmatter?.tags || targetCache?.frontmatter?.tag;
        let targetAllTags = [...targetTags];
        if (targetFmData) {
            if (Array.isArray(targetFmData)) {
                targetAllTags.push(...targetFmData.map(t => String(t).toLowerCase().replace("#", "")));
            } else if (typeof targetFmData === "string") {
                targetAllTags.push(...targetFmData.split(/,\s*/).map(t => t.toLowerCase().replace("#", "")));
            }
        }
        
        const targetNormalizedTags = targetAllTags.map(t => t === "sourcenote" ? "source" : t);
        const isTargetSourceNoteType = VALID_TYPES.map(t => t === "sourcenote" ? "source" : t).some(type => targetNormalizedTags.includes(type));
        if (!isTargetSourceNoteType) {
            validOutgoingLinks++;
            continue;
        }
        
        const targetFrontmatter = targetCache?.frontmatter;
        if (targetFrontmatter && targetFrontmatter.hasOwnProperty("status")) {
            const tStatus = String(targetFrontmatter["status"]).toLowerCase();
            if (tStatus === "linked" || tStatus === "ghost" || tStatus === "orphan") {
                continue; 
            }
        }
        
        validOutgoingLinks++;
    }
}

const backlinksCache = app.metadataCache.getBacklinksForFile(activeFile);
let validIngoingLinks = 0;

if (backlinksCache) {
    const sourcePaths = Array.from(backlinksCache.keys());
    for (const sourcePath of sourcePaths) {
        const sourceFile = app.vault.getAbstractFileByPath(sourcePath);
        if (!sourceFile) continue;
        
        const sourceCache = app.metadataCache.getFileCache(sourceFile);
        const sourceTags = sourceCache?.tags?.map(t => t.tag.toLowerCase().replace("#", "")) || [];
        const sourceFmData = sourceCache?.frontmatter?.tags || sourceCache?.frontmatter?.tag;
        let sourceAllTags = [...sourceTags];
        if (sourceFmData) {
            if (Array.isArray(sourceFmData)) {
                sourceAllTags.push(...sourceFmData.map(t => String(t).toLowerCase().replace("#", "")));
            } else if (typeof sourceFmData === "string") {
                sourceAllTags.push(...sourceFmData.split(/,\s*/).map(t => t.toLowerCase().replace("#", "")));
            }
        }
        
        const sourceNormalizedTags = sourceAllTags.map(t => t === "sourcenote" ? "source" : t);
        const isSourceNoteType = VALID_TYPES.map(t => t === "sourcenote" ? "source" : t).some(type => sourceNormalizedTags.includes(type));
        if (!isSourceNoteType) {
            validIngoingLinks++;
            continue;
        }
        
        const sourceFrontmatter = sourceCache?.frontmatter;
        if (sourceFrontmatter && sourceFrontmatter.hasOwnProperty("status")) {
            const sStatus = String(sourceFrontmatter["status"]).toLowerCase();
            if (sStatus === "linked" || sStatus === "ghost" || sStatus === "orphan") {
                continue;
            }
        }
        
        validIngoingLinks++;
    }
}

const matrix = {
    source: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => validOutgoingLinks >= 2 && wordCount >= 15 },
        populated: { check: () => validOutgoingLinks >= 3 && wordCount >= 50 },
        detailed:  { check: () => validOutgoingLinks >= 4 && wordCount >= 75 }
    },
    jotdown: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => wordCount >= 5 },
        populated: { check: () => wordCount >= 15 },
        detailed:  { check: () => validOutgoingLinks >= 1 && wordCount >= 30 }
    },
    concept: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => validOutgoingLinks >= 2 && wordCount >= 25 },
        populated: { check: () => validOutgoingLinks >= 3 && wordCount >= 60 },
        detailed:  { check: () => validOutgoingLinks >= 4 && wordCount >= 100 }
    },
    personal: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => wordCount >= 5 },
        populated: { check: () => wordCount >= 15 },
        detailed:  { check: () => validOutgoingLinks >= 1 && wordCount >= 30 }
    },
    evergreen: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => validOutgoingLinks >= 2 && wordCount >= 25 },
        populated: { check: () => validOutgoingLinks >= 4 && wordCount >= 60 },
        detailed:  { check: () => validOutgoingLinks >= 5 && wordCount >= 100 }
    },
    idea: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => validOutgoingLinks >= 1 && wordCount >= 20 },
        populated: { check: () => validOutgoingLinks >= 2 && wordCount >= 40 },
        detailed:  { check: () => validOutgoingLinks >= 2 && wordCount >= 60 }
    },
    review: {
        ghost:     { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 && wordCount === 0 },
        orphan:    { check: () => validOutgoingLinks === 0 && validIngoingLinks === 0 },
        linked:    { check: () => (validOutgoingLinks >= 1 || validIngoingLinks >= 1) },
        updated:   { check: () => validOutgoingLinks >= 2 && wordCount >= 25 },
        populated: { check: () => validOutgoingLinks >= 2 && wordCount >= 60 },
        detailed:  { check: () => validOutgoingLinks >= 3 && wordCount >= 100 }
    }
};

const typeRules = matrix[activeType];
const tiers = ["ghost", "orphan", "linked", "updated", "populated", "detailed"];
let statusStr = "ghost";

for (const tier of tiers) {
    if (typeRules[tier] && typeRules[tier].check()) {
        statusStr = tier;
    }
}

await app.fileManager.processFrontMatter(activeFile, (frontmatter) => {
    frontmatter["status"] = statusStr.toLowerCase();
    frontmatter["wordCount"] = wordCount;
    frontmatter["links"] = validOutgoingLinks;
    frontmatter["backLinks"] = validIngoingLinks;
});

return "";
-%>
