# Reddit Newsletter N8N Workflow

A basic N8N automation workflow that demonstrates how to create automated newsletters from Reddit content with AI analysis.

## Overview

This is primarily an educational/demonstration project showing the foundational workflow for building Reddit-based newsletters. The workflow automatically:
- Fetches trending posts from multiple subreddits (r/FantasyPL, r/PremierLeague)
- Filters content based on engagement and relevance
- Generates AI-powered analysis via OpenRouter
- Creates formatted HTML newsletters
- Sends via email

**Note**: This is a basic framework. The AI analysis may contain inaccuracies and should be verified independently. Future iterations will include better fact-checking and data validation.

## Related Projects

This builds on similar automation concepts from [oura-journal-ai-insights](https://github.com/raym26/oura-journal-ai-insights), which demonstrates AI-powered health data analysis workflows.

## Known Limitations

- **AI Accuracy**: The AI analysis may contain factual errors or outdated information about players, transfers, or team rosters. Use as general guidance only.
- **Basic Filtering**: Content filtering is rule-based and may miss nuanced content or include irrelevant posts.
- **No Fact-Checking**: The workflow doesn't verify claims made in Reddit posts or AI analysis.
- **Static Configuration**: Subreddits and filters are hardcoded and require manual updates.

This project serves as a foundation for building more sophisticated newsletter automation systems.

## Workflow Structure

```
Schedule Trigger
    ↓
Reddit (r/FantasyPL) ────┐
                         ├─ Merge
Reddit (r/PremierLeague) ┘    ↓
                         Filter (7-day posts + quality criteria)
                              ↓
                         Sort (by engagement)
                              ↓
                         Extract & Format
                              ↓
                         AI Analysis (OpenRouter)
                              ↓
                         HTML Generation
                              ↓
                         Gmail Send
```

## Setup Instructions

### Prerequisites

1. **N8N Instance**: Self-hosted or N8N Cloud
2. **Reddit Account**: For accessing Reddit API (optional but recommended for higher rate limits)
3. **OpenRouter API Key**: For AI analysis functionality
4. **Gmail Account**: For sending newsletters

### Configuration Steps

#### 1. Reddit Nodes Setup

**Reddit Node 1 (FantasyPL):**
- Resource: Post
- Operation: Get All
- Subreddit: `FantasyPL`
- Category: `New`
- Limit: `100-200`

**Reddit Node 2 (PremierLeague):**
- Resource: Post
- Operation: Get All
- Subreddit: `PremierLeague`
- Category: `New`
- Limit: `100-200`

#### 2. Filter Node Configuration

Use Expression mode with this filter:

```javascript
$json.created_utc > ($now.toUnixInteger() - 604800) &&
(
  $json.title.toLowerCase().includes('injury') || 
  $json.title.toLowerCase().includes('news') || 
  $json.title.toLowerCase().includes('update') || 
  $json.title.toLowerCase().includes('confirmed') || 
  $json.title.toLowerCase().includes('transfer') ||
  $json.title.toLowerCase().includes('signing') ||
  $json.link_flair_text === 'News' ||
  ($json.selftext && parseInt($json.selftext.length) > 200) ||
  parseInt($json.score) > 50
)
```

#### 3. Extract & Format Code Node

```javascript
let newsletterContent = '';
let postCount = 0;

// Process each Reddit post
for (const item of $input.all()) {
  const post = item.json;
  postCount++;
  
  // Skip if we have enough posts (limit to top 10)
  if (postCount > 10) break;
  
  // Determine post type
  let postType = 'Discussion';
  if (post.link_flair_text === 'News') postType = 'News';
  if (post.link_flair_text === 'Statistics') postType = 'Stats';
  if (post.title.toLowerCase().includes('injury')) postType = 'Injury Update';
  if (post.title.toLowerCase().includes('transfer')) postType = 'Transfer News';
  
  // Get content preview
  let content = '';
  if (post.selftext && post.selftext.length > 0) {
    content = post.selftext.substring(0, 300);
    if (post.selftext.length > 300) content += '...';
  } else if (post.post_hint === 'image') {
    content = 'Image post - view on Reddit for full content';
  } else if (post.is_video) {
    content = 'Video post - view on Reddit to watch';
  } else {
    content = 'Link post - click through to Reddit for details';
  }
  
  // Clean up content (remove markdown, HTML)
  content = content.replace(/&lt;[^&gt;]*&gt;/g, ''); // Remove HTML tags
  content = content.replace(/\*\*/g, ''); // Remove bold markdown
  content = content.replace(/\*/g, ''); // Remove italic markdown
  content = content.replace(/\n+/g, ' '); // Replace line breaks with spaces
  content = content.replace(/\s+/g, ' ').trim(); // Clean up spacing
  
  // Format the post
  newsletterContent += `
## ${postCount}. ${post.title}
**Type:** ${postType} | **Score:** ${post.score} upvotes | **Comments:** ${post.num_comments}
**Author:** u/${post.author} in r/${post.subreddit}

${content}

[Read full discussion on Reddit](https://reddit.com${post.permalink})

---

`;
}

// Create summary stats
const totalUpvotes = $input.all().slice(0, 10).reduce((sum, item) => sum + item.json.score, 0);
const totalComments = $input.all().slice(0, 10).reduce((sum, item) => sum + item.json.num_comments, 0);

return {
  newsletter_content: newsletterContent,
  post_count: postCount,
  total_upvotes: totalUpvotes,
  total_comments: totalComments,
  generated_date: new Date().toISOString(),
  subreddit: $input.first().json.subreddit
};
```

#### 4. AI Analysis Setup

**AI Preparation Code Node:**
```javascript
// Get the clean data from Extract and Format node
const data = $input.first().json;
const content = data.newsletter_content;

const analysisPrompt = `As a fantasy football analyst, analyze these trending posts and provide strategic insights:

${content}

Focus on:
- Key takeaways for fantasy managers
- Player value implications
- Transfer recommendations
- Gameweek strategy impact

Keep insights concise and actionable.`;

return { 
  chatInput: analysisPrompt,
  original_data: data
};
```

**AI Agent Node Configuration:**
- LLM Type: HTTP Request LLM
- Base URL: `https://openrouter.ai/api/v1`
- API Key: Your OpenRouter API key
- Model: `anthropic/claude-3-haiku` or `meta-llama/llama-3.1-8b-instruct`
- System Message: "You are a fantasy football analyst specializing in Premier League. Provide strategic insights for fantasy managers focusing on immediate actionable advice."

#### 5. HTML Generation Code Node

```javascript
// Get data from both previous nodes
const originalData = $('Extract and Format').first().json;
const aiAnalysis = $('AI Agent').first().json.output;

const content = originalData.newsletter_content;
const postCount = originalData.post_count;
const totalUpvotes = originalData.total_upvotes;
const totalComments = originalData.total_comments;
const date = new Date().toLocaleDateString();

const htmlNewsletter = `
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Multi-Subreddit Newsletter - ${date}</title>
    <style>
        body { 
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; 
            max-width: 600px; 
            margin: 0 auto; 
            padding: 20px; 
            background-color: #f8f9fa;
            color: #333;
        }
        .container { 
            background: white; 
            padding: 30px; 
            border-radius: 8px; 
            box-shadow: 0 2px 4px rgba(0,0,0,0.1); 
        }
        .header {
            text-align: center;
            border-bottom: 2px solid #ff4500;
            padding-bottom: 20px;
            margin-bottom: 30px;
        }
        h1 { 
            color: #ff4500; 
            margin-bottom: 10px;
            font-size: 28px;
        }
        .subtitle { 
            color: #666; 
            margin-bottom: 20px; 
            font-size: 16px;
        }
        .stats {
            background: #f8f9fa;
            padding: 15px;
            border-radius: 6px;
            margin-bottom: 30px;
            text-align: center;
        }
        .stats-item {
            display: inline-block;
            margin: 0 15px;
            font-weight: 600;
        }
        .summary {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 30px;
            border-left: 4px solid #ff4500;
        }
        .headlines-list {
            margin: 15px 0;
            padding-left: 20px;
        }
        .headlines-list li {
            margin-bottom: 8px;
            line-height: 1.4;
            color: #333;
            font-weight: 500;
        }
        .ai-analysis {
            background: #fff8f0;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 30px;
            border-left: 4px solid #ff6b35;
        }
        .ai-analysis h2 {
            color: #ff6b35;
            margin-top: 0;
        }
        .ai-content {
            line-height: 1.6;
            color: #444;
        }
        h2 { 
            color: #333; 
            border-bottom: 2px solid #ff4500; 
            padding-bottom: 8px;
            margin-top: 30px;
            font-size: 20px;
        }
        .post-meta { 
            color: #666; 
            font-size: 14px; 
            margin-bottom: 15px;
            background: #f8f9fa;
            padding: 10px;
            border-radius: 4px;
        }
        .post-content { 
            line-height: 1.6; 
            margin: 15px 0 20px 0;
            font-size: 15px;
        }
        .post-link { 
            display: inline-block;
            background: #ff4500;
            color: white;
            padding: 8px 16px;
            text-decoration: none;
            border-radius: 4px;
            font-weight: 500;
            margin-bottom: 20px;
        }
        .post-link:hover { 
            background: #e63e00;
            color: white;
        }
        hr { 
            border: none; 
            height: 1px; 
            background: #eee; 
            margin: 25px 0; 
        }
        .footer { 
            margin-top: 40px; 
            text-align: center; 
            color: #666; 
            font-size: 14px;
            border-top: 1px solid #eee;
            padding-top: 20px;
        }
        .reddit-orange { color: #ff4500; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Multi-Subreddit Newsletter</h1>
            <div class="subtitle">${date} • Top ${postCount} trending posts</div>
        </div>
        
        <div class="stats">
            <span class="stats-item reddit-orange">${totalUpvotes} Total Upvotes</span>
            <span class="stats-item reddit-orange">${totalComments} Total Comments</span>
            <span class="stats-item reddit-orange">${postCount} Posts</span>
        </div>
        
        <div class="summary">
            <h2 style="margin-top: 0;">Today's Headlines</h2>
            <ol class="headlines-list">
${content.split('\n## ').slice(1).map((section) => {
  const titleMatch = section.match(/^(\d+)\. (.+?)$/m);
  if (titleMatch) {
    const title = titleMatch[2];
    return `                <li>${title}</li>`;
  }
  return '';
}).filter(item => item).join('')}
            </ol>
        </div>
        
        <div class="ai-analysis">
            <h2>AI Fantasy Analysis</h2>
            <div class="ai-content">
${aiAnalysis.replace(/\n/g, '<br>').replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')}
            </div>
        </div>
        
        <div class="content">
            <h2 style="margin-top: 0;">Full Posts</h2>
${content
  .replace(/## (\d+)\. (.*)/g, '<h2>$1. $2</h2>')
  .replace(/\*\*Type:\*\* (.*?) \| \*\*Score:\*\* (.*?) \| \*\*Comments:\*\* (.*?)\n\*\*Author:\*\* (.*?) in (.*?)\n/g, 
    '<div class="post-meta">Type: $1 • Score: $2 • Comments: $3<br>Author: $4 • Subreddit: $5</div>')
  .replace(/\[Read full discussion on Reddit\]\((.*?)\)/g, 
    '<a href="$1" class="post-link" target="_blank">Read on Reddit</a>')
  .replace(/---/g, '<hr>')
  .replace(/\n\n/g, '</div><div class="post-content">')
}
        </div>
        
        <div class="footer">
            <p><strong>Generated automatically by N8N Reddit Newsletter Bot</strong></p>
            <p>Powered by Reddit API • Enhanced with AI Analysis</p>
        </div>
    </div>
</body>
</html>
`;

return { 
  html_newsletter: htmlNewsletter,
  subject_line: `Multi-Subreddit Newsletter - ${date} (${postCount} trending posts + AI Analysis)`,
  preview_text: `${totalUpvotes} upvotes across ${postCount} top posts with AI fantasy insights`
};
```

#### 6. Gmail Configuration

- Resource: Message
- Operation: Send
- To: Your email address or subscriber list
- Subject: `{{ $json.subject_line }}`
- Email Type: HTML
- Message: `{{ $json.html_newsletter }}`

## Environment Variables

Set these in your N8N environment:

```
OPENROUTER_API_KEY=your_openrouter_api_key
GMAIL_CLIENT_ID=your_gmail_oauth_client_id
GMAIL_CLIENT_SECRET=your_gmail_oauth_client_secret
```

## Customization Options

### Adding More Subreddits

1. Add additional Reddit nodes
2. Connect to the Merge node
3. Update filter criteria if needed

### Modifying Content Filters

Adjust the filter expression to include/exclude different types of content:
- Change minimum score thresholds
- Add/remove keywords
- Modify time windows (currently 7 days)

### AI Analysis Customization

- Change AI model in OpenRouter (claude-3-haiku, llama-3.1-8b-instruct, etc.)
- Modify system prompts for different analysis styles
- Add web search tools for more contextual analysis

### Scheduling Options

- **Daily**: Every day at 8 AM
- **Weekly**: Every Monday at 9 AM
- **Bi-weekly**: Every other Monday
- **Custom**: Based on your preferred schedule

## Troubleshooting

### Common Issues

**Reddit API Rate Limits:**
- Set up Reddit OAuth2 for higher limits
- Reduce the number of posts fetched
- Add delays between requests

**Filter Not Working:**
- Check field names match exactly (`$json.score` vs `score`)
- Use `parseInt()` for numeric comparisons
- Verify date calculations with `$now.toUnixInteger()`

**AI Analysis Fails:**
- Verify OpenRouter API key is correct
- Check model availability and pricing
- Add error handling with fallback content

**Email Not Sending:**
- Verify Gmail OAuth2 setup
- Check email address format
- Test with simple text email first

## Future Improvements

Planned enhancements for this workflow:

- **Fact-checking mechanisms**: Integration with reliable data sources for player/team verification
- **Enhanced AI prompting**: Better guardrails to prevent hallucinations and improve accuracy
- **Content categorization**: Automatic sorting of news vs discussion vs analysis posts
- **Subscriber management**: Database integration for mailing list management
- **Template customization**: Multiple newsletter formats and styling options
- **Real-time data integration**: API connections to FPL official data, transfer APIs
- **Duplicate detection**: Prevent same stories from multiple sources appearing twice

## Contributing

This is a learning project demonstrating N8N automation concepts. Feel free to:
- Submit issues for bugs or improvement suggestions
- Fork and enhance for your own use cases
- Share your workflow modifications
- Add better error handling or data validation

Areas where contributions would be particularly valuable:
- Improved AI prompting techniques
- Better content filtering algorithms
- Additional data source integrations
- Enhanced email template designs

## License

MIT License - feel free to use and modify for your own projects.

## Credits

Built with N8N automation platform, Reddit API, OpenRouter AI services, and Gmail integration.
