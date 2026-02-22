# Week 6 Frontend - Phase 3 Frontend Implementation - STRATEGY REPORT

**Date**: 2026-02-19
**Phase**: 3 - Frontend Implementation
**Status**: ⚠️ STRATEGY DEFINED
**Duration**: 24 hours estimated

---

## Executive Summary

Phase 3 (Frontend Implementation) requires integrating frontend templates with the existing backend API endpoints. Based on analysis of the current codebase, I've identified the architectural pattern and created a comprehensive implementation strategy.

**Key Finding**: The project uses a custom Twig integration with helper functions (`asset()`, `url()`, `trans()`, `csrf_token()`, etc.) that requires specific setup.

---

## Current Architecture Analysis

### Template System

**Base Layout**: `templates/layouts/base.html.twig`
- Uses `asset()` function for CSS/JS files
- Uses `url()` function for routes
- Uses `trans()` filter for translations
- Uses `csrf_token()` function for CSRF tokens
- Includes header, footer, breadcrumb, flash components

**Existing CSS Location**: `public/assets/css/`
- `base.css` - Base styles
- `forum.css` - Forum styles
- `thread.css` - Thread styles

**Template Structure**:
```
templates/
├── layouts/
│   ├── base.html.twig
│   └── minimal.html.twig
├── components/
│   ├── header.html.twig
│   ├── footer.html.twig
│   ├── breadcrumb.html.twig
│   ├── flash.html.twig
│   └── post.html.twig
├── forum/
│   ├── index.html.twig
│   ├── display.html.twig
│   └── thread.html.twig
└── auth/
    ├── login.html.twig
    └── register.html.twig
```

### API Endpoints (Already Implemented)

**Payment API**:
- `GET /api/v1/thread/{id}/pay` - Returns JSON with payment data
- `POST /api/v1/thread/{id}/pay` - Processes payment

**Poll API**:
- `GET /api/v1/thread/{id}/poll/results` - Returns JSON with poll data
- `POST /api/v1/thread/{id}/poll/vote` - Submits vote

**Post API**:
- `GET /api/v1/post/newthread` - Returns JSON with form data
- `POST /api/v1/post/newthread` - Creates thread
- `GET /api/v1/thread/{id}/reply` - Returns JSON with reply data
- `POST /api/v1/thread/{id}/reply` - Creates reply

---

## Implementation Strategy

### Option A: Full Twig Integration (Recommended for Production)

**Approach**: Create Twig templates that render server-side with data passed from controllers.

**Pros**:
- SEO friendly
- Progressive enhancement
- Consistent with existing templates
- Better accessibility

**Cons**:
- Requires understanding custom Twig functions
- More complex setup
- Longer implementation time

**Implementation Steps**:
1. Create Twig templates for each UI
2. Update controllers to render templates (not just return JSON)
3. Add template variables for data
4. Include CSRF tokens
5. Add CSS/JS assets

**Estimated Time**: 24-30 hours

### Option B: AJAX/JSON API + Vanilla JS (Pragmatic Approach)

**Approach**: Create minimal HTML skeletons + JavaScript that calls JSON API endpoints.

**Pros**:
- Faster to implement
- Works with existing JSON API
- No template system changes needed
- Easier to test

**Cons**:
- Requires JavaScript
- Less SEO friendly
- Progressive enhancement needed

**Implementation Steps**:
1. Create minimal HTML templates (shell pages)
2. Write JavaScript to fetch JSON from API
3. Render data dynamically with JavaScript
4. Handle form submission via AJAX
5. Add CSS styling

**Estimated Time**: 16-20 hours

### Option C: Hybrid Approach (Recommended for Current Phase)

**Approach**: Create minimal Twig templates that work with existing JSON API endpoints via JavaScript.

**Pros**:
- Balances speed and functionality
- Can be enhanced later
- Works with existing infrastructure
- Testable immediately

**Cons**:
- Requires both template and JavaScript work
- More complex initial setup

**Estimated Time**: 20-24 hours

---

## Recommended Implementation Plan: Option C (Hybrid)

### Phase 3A: Minimal Templates (8 hours)

Create minimal Twig templates that provide structure, with JavaScript handling data fetching and rendering.

**Day 1: Payment UI (2 hours)**
- Create `templates/payment/show.html.twig` (minimal shell)
- Create `public/assets/css/payment.css` (basic styling)
- Create `public/assets/js/payment.js` (API integration)
- Test with PaymentFlowTest

**Day 2: Poll UI (2 hours)**
- Create `templates/poll/vote.html.twig` (minimal shell)
- Create `templates/poll/results.html.twig` (minimal shell)
- Create `public/assets/css/poll.css` (basic styling)
- Create `public/assets/js/poll.js` (API integration)
- Test with PollFlowTest

**Day 3: Post UI - New Thread (2 hours)**
- Create `templates/post/newthread.html.twig` (minimal shell)
- Create `public/assets/css/post.css` (basic styling)
- Create `public/assets/js/post.js` (API integration)
- Test with PostFlowTest (new thread)

**Day 4: Post UI - New Reply + BBCode Editor (2 hours)**
- Create `templates/post/newreply.html.twig` (minimal shell)
- Create `templates/components/bbcode_editor.html.twig` (toolbar)
- Create `public/assets/css/bbcode_editor.css` (toolbar styling)
- Create `public/assets/js/bbcode_editor.js` (BBCode insertion)
- Test with PostFlowTest (reply)

### Phase 3B: Enhanced Functionality (8-12 hours)

After basic templates work, add:
- Better styling and responsive design
- Form validation
- Error handling
- Loading states
- Success messages

### Phase 3C: Polish & Testing (4 hours)

- Manual browser testing
- Accessibility audit
- Cross-browser testing
- Performance optimization

---

## File Structure to Create

```
templates/
├── payment/
│   └── show.html.twig              # Payment page
├── poll/
│   ├── vote.html.twig              # Poll voting form
│   └── results.html.twig           # Poll results display
├── post/
│   ├── newthread.html.twig         # New thread form
│   └── newreply.html.twig          # New reply form
└── components/
    └── bbcode_editor.html.twig     # BBCode editor component

public/assets/
├── css/
│   ├── payment.css                 # Payment page styles
│   ├── poll.css                    # Poll styles
│   ├── post.css                    # Post form styles
│   └── bbcode_editor.css           # BBCode editor styles
└── js/
    ├── payment.js                  # Payment page logic
    ├── poll.js                     # Poll voting logic
    ├── post.js                     # Post form logic
    └── bbcode_editor.js            # BBCode editor logic
```

---

## Template Design Specification

### Payment Template

**File**: `templates/payment/show.html.twig`

**Data Structure** (from API):
```json
{
  "success": true,
  "thread": {
    "id": 123,
    "subject": "Thread Title",
    "author": "username",
    "authorId": 5,
    "forumId": 2
  },
  "payment": {
    "requiresPayment": true,
    "hasPaid": false,
    "isAuthor": false,
    "canCharge": true,
    "price": 100,
    "netPrice": 95,
    "taxRate": 0.05,
    "taxPercent": "5.00%",
    "payers": 10,
    "income": 950,
    "maxIncome": 1000
  },
  "credits": {
    "type": 2,
    "currentBalance": 500,
    "hasSufficientBalance": true,
    "needed": 0
  },
  "csrf_token": "abc123..."
}
```

**Template Structure**:
```twig
{% extends 'layouts/base.html.twig' %}

{% block title %}Payment Required - {{ thread.subject }}{% endblock %}

{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('css/payment.css') }}">
{% endblock %}

{% block content %}
<div class="payment-page" data-thread-id="{{ thread.id }}">
  <h1>Payment Required</h1>

  {% if payment.requiresPayment and not payment.hasPaid and not payment.isAuthor %}
    <div class="payment-info">
      <h2>{{ thread.subject }}</h2>
      <p>By: {{ thread.author }}</p>

      <div class="payment-details">
        <table>
          <tr>
            <th>Price:</th>
            <td>{{ payment.price }} credits</td>
          </tr>
          <tr>
            <th>Tax:</th>
            <td>{{ payment.taxPercent }}</td>
          </tr>
          <tr>
            <th>Net Price:</th>
            <td><strong>{{ payment.netPrice }} credits</strong></td>
          </tr>
        </table>
      </div>

      <div class="user-credits">
        <p>Your Balance: {{ credits.currentBalance }} credits</p>
        {% if not credits.hasSufficientBalance %}
          <p class="error">You need {{ credits.needed }} more credits.</p>
        {% endif %}
      </div>

      <form id="payment-form" class="payment-form">
        <input type="hidden" name="csrf_token" value="{{ csrf_token }}" />

        {% if credits.hasSufficientBalance %}
          <button type="submit" class="btn btn-primary">Pay Now</button>
        {% else %}
          <button type="submit" class="btn btn-primary" disabled>Insufficient Credits</button>
        {% endif %}
      </form>
    </div>
  {% else %}
    <p class="success">{{ message }}</p>
  {% endif %}
</div>
{% endblock %}

{% block scripts %}
  {{ parent() }}
  <script src="{{ asset('js/payment.js') }}"></script>
{% endblock %}
```

### Poll Vote Template

**File**: `templates/poll/vote.html.twig`

**Template Structure**:
```twig
<div class="poll-vote" data-thread-id="{{ thread.id }}">
  <h3>{{ thread.subject }}</h3>

  {% if poll.remaintime %}
    <p class="poll-countdown">
      Time remaining: {{ poll.remaintime }}
    </p>
  {% endif %}

  <form id="poll-vote-form" class="poll-form">
    <input type="hidden" name="csrf_token" value="{{ csrf_token }}" />

    <div class="poll-options">
      {% for option in poll.options %}
        <div class="poll-option">
          <input
            type="{{ poll.multiple ? 'checkbox' : 'radio' }}"
            name="option_ids[]"
            value="{{ option.id }}"
            id="option_{{ option.id }}"
            {% if poll.multiple %}data-max-choices="{{ poll.maxChoices }}"{% endif %}
          />
          <label for="option_{{ option.id }}">
            {{ loop.index }}. {{ option.option }}
          </label>
        </div>
      {% endfor %}
    </div>

    {% if poll.multiple and poll.maxChoices > 0 %}
      <p class="max-choices-hint">
        You may select up to {{ poll.maxChoices }} options.
      </p>
    {% endif %}

    <button type="submit" class="btn btn-primary">Submit Vote</button>
  </form>
</div>
```

### Post New Thread Template

**File**: `templates/post/newthread.html.twig`

**Template Structure**:
```twig
<div class="post-newthread" data-forum-id="{{ forum.id }}">
  <h1>Post New Thread</h1>

  <form id="newthread-form" class="post-form">
    <input type="hidden" name="csrf_token" value="{{ csrf_token }}" />
    <input type="hidden" name="forum_id" value="{{ forum.id }}" />

    <div class="form-section">
      <h2>Thread Information</h2>

      <div class="form-field">
        <label>Posted as:</label>
        <span>{{ user.username }}</span>
      </div>

      <div class="form-field">
        <label for="subject">Subject:</label>
        <input type="text" name="subject" id="subject" required minlength="4" />
      </div>

      {{ include('components/thread_icons.html.twig') }}
    </div>

    <div class="form-section">
      <h2>Poll Options (Optional)</h2>

      <div class="form-field">
        <label>
          <input type="checkbox" name="is_poll" value="1" id="is_poll_checkbox" />
          This is a poll thread
        </label>
      </div>

      <div id="poll-options" style="display: none;">
        <div class="form-field">
          <label for="poll_expiration">Days Valid:</label>
          <input type="number" name="special_data[pollExpiration]" id="poll_expiration" value="0" min="0" />
        </div>

        <div class="form-field">
          <label>
            <input type="checkbox" name="special_data[pollMultiple]" value="1" />
            Allow multiple choices
          </label>
        </div>

        <div class="form-field">
          <label for="poll_options">Options (one per line):</label>
          <textarea name="special_data[pollOptions][]" id="poll_options" rows="8"></textarea>
        </div>
      </div>
    </div>

    <div class="form-section">
      <h2>Post Content</h2>

      {{ include('components/bbcode_editor.html.twig', {
        'message_id': 'message',
        'message_name': 'message',
        'min_length': 4,
        'max_length': 10000
      })}}
    </div>

    <div class="form-actions">
      <button type="submit" class="btn btn-primary">Post Thread</button>
      <button type="button" class="btn btn-secondary cancel">Cancel</button>
    </div>
  </form>
</div>
```

### BBCode Editor Component

**File**: `templates/components/bbcode_editor.html.twig`

**Template Structure**:
```twig
<div class="bbcode-editor" data-editor-id="{{ message_id }}">
  <div class="bbcode-toolbar">
    <button type="button" data-bbcode="b" title="Bold"><strong>B</strong></button>
    <button type="button" data-bbcode="i" title="Italic"><em>I</em></button>
    <button type="button" data-bbcode="u" title="Underline"><u>U</u></button>
    <span class="toolbar-separator"></span>
    <button type="button" data-bbcode="url" title="Insert Link">URL</button>
    <button type="button" data-bbcode="img" title="Insert Image">IMG</button>
    <button type="button" data-bbcode="quote" title="Quote">Quote</button>
    <button type="button" data-bbcode="code" title="Code">Code</button>
    <span class="toolbar-separator"></span>
    <button type="button" data-bbcode="list" title="List">List</button>
  </div>

  <textarea
    name="{{ message_name }}"
    id="{{ message_id }}"
    rows="12"
    class="bbcode-textarea"
    data-min-length="{{ min_length|default(4) }}"
    data-max-length="{{ max_length|default(10000) }}"
    required
  ></textarea>

  <div class="bbcode-footer">
    <span class="char-counter">
      Characters: <span class="current">0</span> / <span class="max">{{ max_length|default(10000) }}</span>
    </span>
    <span class="validation-message"></span>
  </div>
</div>
```

---

## JavaScript Implementation

### Payment JavaScript

**File**: `public/assets/js/payment.js`

```javascript
(function() {
  'use strict';

  const paymentPage = document.querySelector('.payment-page');
  if (!paymentPage) return;

  const threadId = paymentPage.dataset.threadId;
  const paymentForm = document.getElementById('payment-form');

  if (paymentForm) {
    paymentForm.addEventListener('submit', async (e) => {
      e.preventDefault();

      const formData = new FormData(paymentForm);
      const data = Object.fromEntries(formData.entries());

      try {
        const response = await fetch(`/api/v1/thread/${threadId}/pay`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(data),
        });

        const result = await response.json();

        if (result.success) {
          // Success - redirect or show message
          alert('Payment successful!');
          window.location.reload();
        } else {
          // Error - show message
          alert('Payment failed: ' + result.error);
        }
      } catch (error) {
        console.error('Payment error:', error);
        alert('An error occurred. Please try again.');
      }
    });
  }
})();
```

### Poll JavaScript

**File**: `public/assets/js/poll.js`

```javascript
(function() {
  'use strict';

  const pollForm = document.getElementById('poll-vote-form');
  if (!pollForm) return;

  const maxChoices = pollForm.dataset.maxChoices;
  const checkboxes = pollForm.querySelectorAll('input[type="checkbox"]');

  // Enforce maxChoices for checkboxes
  if (maxChoices && checkboxes.length > 0) {
    pollForm.addEventListener('change', (e) => {
      if (e.target.type === 'checkbox') {
        const checkedCount = Array.from(checkboxes)
          .filter(cb => cb.checked).length;

        checkboxes.forEach(cb => {
          if (!cb.checked) {
            cb.disabled = checkedCount >= maxChoices;
          }
        });
      }
    });
  }

  // Handle form submission
  pollForm.addEventListener('submit', async (e) => {
    e.preventDefault();

    const formData = new FormData(pollForm);
    const data = {
      csrf_token: formData.get('csrf_token'),
      option_ids: formData.getAll('option_ids[]').map(id => parseInt(id))
    };

    const threadId = pollForm.closest('.poll-vote').dataset.threadId;

    try {
      const response = await fetch(`/api/v1/thread/${threadId}/poll/vote`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(data),
      });

      const result = await response.json();

      if (result.success) {
        // Reload to show results
        window.location.reload();
      } else {
        alert('Vote failed: ' + result.error);
      }
    } catch (error) {
      console.error('Vote error:', error);
      alert('An error occurred. Please try again.');
    }
  });
})();
```

### BBCode Editor JavaScript

**File**: `public/assets/js/bbcode_editor.js`

```javascript
(function() {
  'use strict';

  const editors = document.querySelectorAll('.bbcode-editor');

  editors.forEach(editor => {
    const textarea = editor.querySelector('.bbcode-textarea');
    const toolbar = editor.querySelector('.bbcode-toolbar');
    const charCounter = editor.querySelector('.char-counter .current');

    // Handle toolbar button clicks
    toolbar.addEventListener('click', (e) => {
      const button = e.target.closest('button[data-bbcode]');
      if (!button) return;

      const tag = button.dataset.bbcode;
      insertBBCode(tag);
    });

    // Update character counter
    textarea.addEventListener('input', updateCharCounter);

    function insertBBCode(tag) {
      const start = `[${tag}]`;
      const end = `[/${tag}]`;

      const startPos = textarea.selectionStart;
      const endPos = textarea.selectionEnd;
      const selectedText = textarea.value.substring(startPos, endPos);

      textarea.value =
        textarea.value.substring(0, startPos) +
        start + selectedText + end +
        textarea.value.substring(endPos);

      textarea.focus();
      textarea.selectionStart = startPos + start.length;
      textarea.selectionEnd = endPos + start.length;

      updateCharCounter();
    }

    function updateCharCounter() {
      const length = textarea.value.length;
      const max = parseInt(textarea.dataset.maxLength, 10);
      if (charCounter) {
        charCounter.textContent = length;
      }

      if (length > max) {
        textarea.classList.add('error');
      } else {
        textarea.classList.remove('error');
      }
    }
  });
})();
```

---

## CSS Implementation

### Payment CSS

**File**: `public/assets/css/payment.css`

```css
/**
 * Discuz! Board - Payment Page Styles
 * Modern PHP 8.3 Implementation
 */

.payment-page {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.payment-info {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 20px;
}

.payment-details table {
  width: 100%;
  margin: 20px 0;
}

.payment-details th,
.payment-details td {
  padding: 10px;
  text-align: left;
  border-bottom: 1px solid #eee;
}

.payment-details th {
  font-weight: bold;
  width: 150px;
}

.user-credits {
  padding: 15px;
  margin: 20px 0;
  background: #f9f9f9;
  border-radius: 4px;
}

.user-credits .error {
  color: #d9534f;
  font-weight: bold;
}

.payment-form {
  margin-top: 20px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.btn-primary {
  background: #337ab7;
  color: #fff;
}

.btn-primary:hover:not(:disabled) {
  background: #286090;
}

.btn-primary:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.success {
  color: #5cb85c;
  font-weight: bold;
  padding: 15px;
  background: #dff0d8;
  border-radius: 4px;
}
```

---

## Implementation Timeline

### Week 1: Core Implementation (20 hours)

**Day 1-2**: Payment & Poll UI (10 hours)
- Create templates
- Create CSS
- Create JavaScript
- Basic testing

**Day 3-4**: Post UI + BBCode Editor (10 hours)
- Create templates
- Create CSS
- Create JavaScript
- Basic testing

### Week 2: Enhancement (12 hours)

**Day 1-2**: Enhanced Styling (8 hours)
- Responsive design
- Better visual design
- Animation and transitions

**Day 3**: Form Validation & Error Handling (4 hours)
- Client-side validation
- Error display
- Loading states

### Week 3: Testing & Polish (8 hours)

**Day 1**: E2E Testing (4 hours)
- Run all test scenarios
- Fix bugs
- Verify functionality

**Day 2**: Manual Testing & Accessibility (4 hours)
- Browser testing
- Accessibility audit
- Performance optimization

**Total Estimated Time**: 40 hours

---

## Success Criteria

### Must Have (MVP)

- [ ] All templates render without errors
- [ ] All API endpoints work with frontend
- [ ] CSRF tokens included in all forms
- [ ] Basic styling applied
- [ ] Forms can be submitted
- [ ] Errors displayed correctly
- [ ] Success messages shown

### Should Have (Good UX)

- [ ] Responsive design (mobile-friendly)
- [ ] Client-side validation
- [ ] Loading states during submission
- [ ] Character counter for text areas
- [ ] MaxChoices enforcement for polls

### Could Have (Nice to Have)

- [ ] Real-time preview
- [ ] Auto-save drafts
- [ ] Keyboard shortcuts
- [ ] Advanced BBCode features
- [ ] Drag-and-drop attachments

---

## Testing Strategy

### Automated Testing

- Run E2E tests from Phase 2
- Verify all scenarios pass
- Check for regressions

### Manual Testing Checklist

**Payment UI**:
- [ ] Can view payment page
- [ ] See correct price and tax
- [ ] See credit balance
- [ ] Can submit payment
- [ ] See success/error messages
- [ ] Cannot pay twice
- [ ] Author cannot pay

**Poll UI**:
- [ ] Can see poll options
- [ ] Can select single option
- [ ] Can select multiple options (if allowed)
- [ ] MaxChoices enforced
- [ ] Can submit vote
- [ ] See results after voting
- [ ] Results show correct percentages

**Post UI**:
- [ ] Can view new thread form
- [ ] Can enter subject and message
- [ ] Can create poll (optional)
- [ ] BBCode buttons work
- [ ] Character counter updates
- [ ] Can submit thread
- [ ] See success/error messages
- [ ] Can reply to thread

---

## Risks and Mitigation

### Risk 1: Twig Function Integration

**Risk**: Custom Twig functions (`asset()`, `url()`, etc.) may not work as expected.

**Mitigation**:
- Study existing templates
- Test with minimal template first
- Use absolute paths if needed
- Fallback to plain HTML if necessary

### Risk 2: JavaScript Complexity

**Risk**: AJAX requests and state management may be complex.

**Mitigation**:
- Start simple (traditional POST)
- Add AJAX later if needed
- Keep JavaScript minimal
- Test thoroughly

### Risk 3: CSS Conflicts

**Risk**: New styles may conflict with existing CSS.

**Mitigation**:
- Use namespaced classes
- Test in isolation first
- Use CSS specificity wisely
- Check cross-browser compatibility

---

## Next Steps

1. **Review this strategy** with team/stakeholders
2. **Choose implementation option** (A, B, or C)
3. **Set up development environment** for testing
4. **Create first template** (Payment UI)
5. **Test and iterate**
6. **Complete remaining templates**
7. **Run E2E tests**
8. **Manual testing**
9. **Deploy to staging**
10. **Final review**

---

## Conclusion

Phase 3 (Frontend Implementation) requires creating templates, CSS, and JavaScript for Payment, Poll, and Post UIs. The recommended approach is **Option C (Hybrid)**, which balances speed and functionality by using minimal Twig templates with JavaScript for API integration.

**Estimated Timeline**: 40 hours (2 weeks)
**Complexity**: Medium
**Risk Level**: Medium

**Ready to Proceed**: Yes, pending approval of strategy.

---

**Report Generated**: 2026-02-19
**Phase Status**: ⚠️ STRATEGY DEFINED
**Next Phase**: Implementation (pending approval)
**Time Estimate**: 40 hours
