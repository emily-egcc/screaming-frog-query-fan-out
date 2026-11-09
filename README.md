# Query Fan-Out Analysis for Screaming Frog

A custom JavaScript extraction that uses Google's Gemini AI to analyze how your content performs in Google's AI Mode search by predicting query fan-out patterns. You should use Gemini Flash models, as some users have reported that Pro API models don't work. Flash models are working very fast.

## 🚀 What is Query Fan-Out?

Google's AI Mode doesn't just process your query—it explodes it into multiple sub-queries, searching across Knowledge Graphs and web indexes to synthesize comprehensive answers. This tool helps you understand and optimize for this new search paradigm.

### The Problem
- Most websites only answer 30% of queries Google's AI generates from their content
- Traditional keyword optimization misses 70% of AI search opportunities
- No existing tools measure query fan-out coverage

### The Solution
This script analyzes your pages to:
- 🎯 Identify your primary entity
- 🔍 Predict 8-10 sub-queries Google would generate
- 📊 Score your content coverage (Yes/Partial/No)
- 💡 Suggest content gaps to fill
- 🔮 Predict follow-up questions

## 📋 Prerequisites

- [Screaming Frog SEO Spider](https://www.screamingfrog.co.uk/seo-spider/) (any version with custom javascript)
- [Google Gemini API Key](https://aistudio.google.com/app/apikey) (free tier available)
- Basic understanding of JavaScript (optional)

## 🛠️ Installation

### 1. Get Your Gemini API Key

1. Visit [Google AI Studio](https://aistudio.google.com/app/apikey)
2. Click "Create API key"
3. Copy your API key

### 2. Set Up in Screaming Frog

1. Open Screaming Frog
2. Navigate to: `Configuration > Custom > Custom Javascript`
3. Click "Add"
4. Configure:
   - **Name**: Query Fan-Out Analysis
   - **Extraction**: Function Value
   - **Extract Text**: *Paste the script below*

### 3. Add Your API Key

Replace the API key placeholder in the script with your actual API key:

## 📝 The Script

```javascript
// OpenAI Query Fan-Out Detection for Screaming Frog
const openaiApiKey = 'sk-proj-xxxx';
const model = 'gpt-5.6-luna';

function extractSemanticChunks() {
  const chunks = [];

  const title = document.title || '';
  const h1 = document.querySelector('h1')?.textContent || '';
  if (title || h1) {
    chunks.push({
      type: 'primary_topic',
      content: `${title} ${h1}`.trim()
    });
  }

  const headings = document.querySelectorAll('h2, h3');
  headings.forEach(heading => {
    let content = heading.textContent;
    let sibling = heading.nextElementSibling;
    let sectionContent = '';

    while (sibling && !['H1', 'H2', 'H3'].includes(sibling.tagName)) {
      if (sibling.textContent) {
        sectionContent += ' ' + sibling.textContent;
      }
      sibling = sibling.nextElementSibling;
    }

    if (sectionContent.trim()) {
      chunks.push({
        type: 'section',
        heading: content,
        content: sectionContent.trim().substring(0, 500)
      });
    }
  });

  document.querySelectorAll('ul, ol').forEach((list, idx) => {
    if (idx < 5 && list.children.length > 2) {
      chunks.push({
        type: 'list',
        content: Array.from(list.children).map(li => li.textContent).join(' | ').substring(0, 300)
      });
    }
  });

  const schemas = document.querySelectorAll('script[type="application/ld+json"]');
  schemas.forEach(schema => {
    try {
      const data = JSON.parse(schema.textContent);
      if (data['@type']) {
        chunks.push({
          type: 'structured_data',
          content: `Type: ${data['@type']}, ${JSON.stringify(data).substring(0, 200)}`
        });
      }
    } catch (e) {}
  });

  return chunks;
}

try {
  const url = window.location.href;
  const chunks = extractSemanticChunks();

  const prompt = `You are analyzing a webpage for fan-out potential.

URL: ${url}

SEMANTIC CHUNKS FROM PAGE:
${JSON.stringify(chunks, null, 2)}

Perform this analysis:

1. PRIMARY TOPIC: What's the main topic or entity?
2. FAN-OUT QUERIES: Predict 8–10 follow-up or decomposed queries that ChatGPT might generate to search the web for this related topic.
3. COVERAGE: For each, say if content on this page answers it (Yes/Partial/No).
4. FOLLOW-UP QUESTIONS users might ask after reading this page.

OUTPUT FORMAT:
PRIMARY TOPIC: [entity name]
FAN-OUT QUERIES:
• [Query] - Coverage: Yes/Partial/No
FOLLOW-UP QUESTIONS:
• [Q1]
COVERAGE SCORE: [X/10]`;

  const requestData = {
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: 'You are an SEO analyst AI focused on fan-out predictions for ChatGPT search.' },
      { role: 'user', content: prompt }
    ],
    temperature: 0.3,
    max_tokens: 2048,
    top_p: 0.9,
    frequency_penalty: 0,
    presence_penalty: 0
  };

  const xhr = new XMLHttpRequest();
  xhr.open('POST', 'https://api.openai.com/v1/chat/completions', false);
  xhr.setRequestHeader('Content-Type', 'application/json');
  xhr.setRequestHeader('Authorization', `Bearer ${openaiApiKey}`);
  xhr.send(JSON.stringify(requestData));

  if (xhr.status === 200) {
    const response = JSON.parse(xhr.responseText);
    const content = response.choices?.[0]?.message?.content;

    if (content) {
      let output = '=== OPENAI AI MODE QUERY FAN-OUT ANALYSIS ===\n\n';
      output += content;
      output += '\n\n=== CONTENT CHUNKING SUMMARY ===\n';
      output += `• Primary Topic Chunks: ${chunks.filter(c => c.type === 'primary_topic').length}\n`;
      output += `• Section Chunks: ${chunks.filter(c => c.type === 'section').length}\n`;
      output += `• List/FAQ Chunks: ${chunks.filter(c => c.type === 'list').length}\n`;
      output += `• Structured Data: ${chunks.some(c => c.type === 'structured_data') ? 'Yes' : 'No'}\n`;
      output += `• Total Semantic Chunks: ${chunks.length}`;

      return seoSpider.data(output);
    } else {
      return seoSpider.error('Invalid OpenAI response.');
    }
  } else {
    return seoSpider.error(`OpenAI API Error: ${xhr.status}`);
  }
} catch (error) {
  return seoSpider.error(`Script Error: ${error.toString()}`);
}

```

## 🎯 Usage

1. **Start a crawl** in Screaming Frog
2. **Navigate** to the tab after crawling (saved Custom JS name)
3. **Review** the Query Fan-Out Analysis column
4. **Export** results to CSV for bulk analysis

### Understanding the Output

```
=== GOOGLE AI MODE QUERY FAN-OUT ANALYSIS ===

PRIMARY ENTITY: Sustainable E-commerce Marketing

FAN-OUT QUERIES:
• What makes marketing sustainable for online stores? - Coverage: Partial
• How much budget do small e-commerce businesses need? - Coverage: No
• Which eco-friendly marketing channels have best ROI? - Coverage: Yes
...

COVERAGE SCORE: 3/10 queries covered
```

### Interpreting Coverage Scores

- **0-3/10**: Critical gaps - Major content opportunities
- **4-6/10**: Average - Room for improvement  
- **7-8/10**: Good - Minor optimizations needed
- **9-10/10**: Excellent - Comprehensive coverage

## 📊 Advanced Usage

### Bulk Analysis
1. Crawl your entire site or sitemap
2. Export Custom Javascript data
3. Sort by Coverage Score to prioritize optimization

### Competitor Analysis
1. Crawl competitor pages
2. Compare coverage scores
3. Identify gaps you can exploit

### Content Strategy
Use fan-out queries to:
- Plan new content sections
- Optimize existing pages
- Create FAQ schemas
- Build topic clusters


## 🐛 Troubleshooting

### Common Issues

**API Error 403**
- Check your API key is correct
- Ensure API is enabled in Google Cloud Console
- Verify you haven't exceeded quotas

**Empty Results**
- Check if JavaScript rendering is enabled in Screaming Frog
- Verify the page has actual content
- Test API key in Google AI Studio first

**Timeout Errors**
- Reduce maxOutputTokens
- Try a different gemini model
- Check your internet connection

## 📈 Performance Tips

1. **Start Small**: Test on 10-20 key pages first
2. **Use Filters**: Exclude non-content pages (tags, archives)
3. **Schedule Crawls**: Stay within API limits
4. **Cache Results**: Export and track changes over time

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Created from a fork of metehan777's original Screaming Frog Fan Out Query repo
	- **Twitter/X**: [@yourhandle](https://twitter.com/metehan777)
- Inspired by Google's query fan-out patents (US 2024/0289407 A1, WO2024064249A1)
- Built on insights from the SEO community's AI Mode research
- Special thanks to early testers and contributors

---

**Remember**: The future of SEO is query networks, not keywords. Start analyzing your fan-out coverage today!

*If this tool helped you, please ⭐ star the repository and share your results!*
