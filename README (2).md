#!/usr/bin/env node
// Renders a meeting summary memo (.docx) from a memo_content.json file
// (see memo_content.schema.json for the shape). Purely local: reads one
// JSON file, writes one .docx file, touches nothing else.
//
// Usage: node build_memo.js <memo_content.json> <output.docx> [draft|final]
'use strict';

const fs = require('fs');
const path = require('path');
const {
  Document, Packer, Paragraph, TextRun, HeadingLevel, Table, TableRow, TableCell,
  WidthType, ShadingType, BorderStyle, AlignmentType, Header, Footer, PageNumber,
  LevelFormat, convertInchesToTwip,
} = require('docx');

const [, , contentPath, outputPath, modeArg] = process.argv;
if (!contentPath || !outputPath) {
  console.error('Usage: build_memo.js <memo_content.json> <output.docx> [draft|final]');
  process.exit(1);
}
const mode = (modeArg || 'draft').toLowerCase();
if (mode !== 'draft' && mode !== 'final') {
  console.error(`Unknown mode "${mode}" — expected "draft" or "final".`);
  process.exit(1);
}

const content = JSON.parse(fs.readFileSync(contentPath, 'utf8'));

const PAGE_WIDTH = 12240; // US Letter, DXA
const PAGE_HEIGHT = 15840;
const MARGIN = 1440; // 1"
const CONTENT_WIDTH = PAGE_WIDTH - MARGIN * 2;

const COL_ITEM = Math.round(CONTENT_WIDTH * 0.5);
const COL_OWNER = Math.round(CONTENT_WIDTH * 0.25);
const COL_TIMELINE = CONTENT_WIDTH - COL_ITEM - COL_OWNER;

function bulletPara(text) {
  return new Paragraph({ numbering: { reference: 'bullets', level: 0 }, children: [new TextRun(text)] });
}

function headerCell(text, width) {
  return new TableCell({
    width: { size: width, type: WidthType.DXA },
    shading: { type: ShadingType.CLEAR, fill: 'DDDDDD' },
    children: [new Paragraph({ children: [new TextRun({ text, bold: true })] })],
  });
}

function bodyCell(text, width) {
  return new TableCell({
    width: { size: width, type: WidthType.DXA },
    children: [new Paragraph({ children: [new TextRun(text || '—')] })],
  });
}

function actionItemsTable(items) {
  const rows = [
    new TableRow({
      tableHeader: true,
      children: [
        headerCell('Action Item', COL_ITEM),
        headerCell('Owner', COL_OWNER),
        headerCell('Timeline', COL_TIMELINE),
      ],
    }),
    ...items.map(
      (i) =>
        new TableRow({
          children: [bodyCell(i.item, COL_ITEM), bodyCell(i.owner, COL_OWNER), bodyCell(i.timeline, COL_TIMELINE)],
        })
    ),
  ];
  return new Table({
    width: { size: CONTENT_WIDTH, type: WidthType.DXA },
    columnWidths: [COL_ITEM, COL_OWNER, COL_TIMELINE],
    rows,
  });
}

const children = [];

if (mode === 'draft') {
  children.push(
    new Paragraph({
      alignment: AlignmentType.CENTER,
      border: { bottom: { style: BorderStyle.SINGLE, size: 6, color: 'C00000' } },
      children: [
        new TextRun({
          text: 'DRAFT — PENDING HUMAN REVIEW. Not for distribution until accepted by the operator.',
          bold: true,
          color: 'C00000',
        }),
      ],
    }),
    new Paragraph({ text: '' })
  );
}

children.push(
  new Paragraph({ text: content.memoTitle || 'Meeting Summary Memo', heading: HeadingLevel.TITLE }),
  new Paragraph({ children: [new TextRun({ text: 'Date: ', bold: true }), new TextRun(content.memoDate || '')] })
);
if (content.preparedFor) {
  children.push(
    new Paragraph({ children: [new TextRun({ text: 'Prepared for: ', bold: true }), new TextRun(content.preparedFor)] })
  );
}
if (content.preparedBy) {
  children.push(
    new Paragraph({ children: [new TextRun({ text: 'Prepared by: ', bold: true }), new TextRun(content.preparedBy)] })
  );
}
children.push(
  new Paragraph({
    children: [
      new TextRun({ text: 'Status: ', bold: true }),
      new TextRun(mode === 'draft' ? 'DRAFT' : 'FINAL — accepted by operator'),
    ],
  }),
  new Paragraph({ text: '' })
);

if (Array.isArray(content.meetings) && content.meetings.length > 1) {
  children.push(new Paragraph({ text: 'Meetings Covered', heading: HeadingLevel.HEADING_1 }));
  for (const m of content.meetings) {
    children.push(bulletPara(`${m.name}${m.date ? ' — ' + m.date : ''}`));
  }
  children.push(new Paragraph({ text: '' }));
}

for (const m of content.meetings || []) {
  children.push(new Paragraph({ text: `${m.name}${m.date ? ' — ' + m.date : ''}`, heading: HeadingLevel.HEADING_1 }));
  if (m.attendees && m.attendees.length) {
    children.push(
      new Paragraph({ children: [new TextRun({ text: 'Attendees: ', bold: true }), new TextRun(m.attendees.join(', '))] })
    );
  }
  if (m.sourceFile) {
    children.push(
      new Paragraph({
        children: [new TextRun({ text: 'Source: ', italics: true }), new TextRun({ text: m.sourceFile, italics: true })],
      })
    );
  }
  children.push(new Paragraph({ text: 'Summary', heading: HeadingLevel.HEADING_2 }));
  children.push(new Paragraph({ text: m.summary || '' }));

  children.push(new Paragraph({ text: 'Action Items', heading: HeadingLevel.HEADING_2 }));
  if (m.actionItems && m.actionItems.length) {
    children.push(actionItemsTable(m.actionItems));
  } else {
    children.push(new Paragraph({ text: 'No action items captured for this meeting.' }));
  }
  children.push(new Paragraph({ text: '' }));

  if (m.marketConnections && m.marketConnections.narrative) {
    children.push(new Paragraph({ text: 'Market Context', heading: HeadingLevel.HEADING_2 }));
    children.push(new Paragraph({ text: m.marketConnections.narrative }));
    if (m.marketConnections.sources && m.marketConnections.sources.length) {
      children.push(
        new Paragraph({
          children: [
            new TextRun({ text: 'Market sources: ', italics: true }),
            new TextRun({ text: m.marketConnections.sources.join(', '), italics: true }),
          ],
        })
      );
    }
    children.push(new Paragraph({ text: '' }));
  }
}

children.push(new Paragraph({ text: 'Suggested Next Steps', heading: HeadingLevel.HEADING_1 }));
children.push(
  new Paragraph({
    text:
      'The following steps connect discussion points across the meetings above with the uploaded market information, so the recommended path forward reflects both internal decisions and what is happening in the market.',
  })
);
for (const s of content.nextSteps || []) {
  children.push(bulletPara(s.step));
  if (s.rationale) {
    children.push(
      new Paragraph({
        indent: { left: convertInchesToTwip(0.5) },
        children: [new TextRun({ text: s.rationale, italics: true })],
      })
    );
  }
}

if (content.openQuestionsForReviewer && content.openQuestionsForReviewer.length) {
  children.push(new Paragraph({ text: '' }));
  children.push(new Paragraph({ text: 'Open Questions for Reviewer', heading: HeadingLevel.HEADING_1 }));
  for (const q of content.openQuestionsForReviewer) {
    children.push(bulletPara(q));
  }
}

children.push(new Paragraph({ text: '' }));
if (mode === 'draft') {
  children.push(
    new Paragraph({
      border: { top: { style: BorderStyle.SINGLE, size: 6, color: 'C00000' } },
      children: [
        new TextRun({
          text:
            'Sign-off required: this draft must be reviewed and explicitly accepted by the operator before a final version is generated. No copy of this memo is sent or shared automatically.',
          bold: true,
          color: 'C00000',
        }),
      ],
    })
  );
} else {
  children.push(
    new Paragraph({
      children: [new TextRun({ text: 'Accepted by operator on: ', bold: true }), new TextRun('_______________________')],
    })
  );
}

const doc = new Document({
  numbering: {
    config: [
      {
        reference: 'bullets',
        levels: [{ level: 0, format: LevelFormat.BULLET, text: '•', alignment: AlignmentType.LEFT }],
      },
    ],
  },
  sections: [
    {
      properties: {
        page: {
          size: { width: PAGE_WIDTH, height: PAGE_HEIGHT },
          margin: { top: MARGIN, bottom: MARGIN, left: MARGIN, right: MARGIN },
        },
      },
      headers: {
        default: new Header({
          children: [
            new Paragraph({
              alignment: AlignmentType.CENTER,
              children: [
                new TextRun({
                  text: mode === 'draft' ? 'DRAFT — PENDING HUMAN REVIEW' : content.memoTitle || 'Meeting Summary Memo',
                  bold: mode === 'draft',
                  color: mode === 'draft' ? 'C00000' : '808080',
                  size: 16,
                }),
              ],
            }),
          ],
        }),
      },
      footers: {
        default: new Footer({
          children: [
            new Paragraph({
              alignment: AlignmentType.CENTER,
              children: [
                new TextRun({ children: ['Page ', PageNumber.CURRENT, ' of ', PageNumber.TOTAL_PAGES], size: 16 }),
              ],
            }),
          ],
        }),
      },
      children,
    },
  ],
});

Packer.toBuffer(doc).then((buf) => {
  fs.mkdirSync(path.dirname(outputPath), { recursive: true });
  fs.writeFileSync(outputPath, buf);
  console.log(`Wrote ${mode} memo to ${outputPath}`);
});
