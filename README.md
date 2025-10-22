# EdTech Impact Analytics API 

 



## Table of Contents


- [Direct API Integration](#direct-api-integration)

- [Event Reference](#event-reference)

- [API Reference](#api-reference)

 
---

## Direct API Integration

  


  

### Authentication

  

Include your API key in the `Authorization` header:

  

```

Authorization: Bearer your-api-key-here

```

  

### Endpoint

  

```

POST https://im.pact.workers.dev/

Content-Type: application/json

Authorization: Bearer your-api-key-here

```

  

### Request Format

  

**Single Event:**

```json

{
	"event": "completed",
	"userId": "12345",
	"traits": {
		"role": "student",
		"schoolId": "sch_oak_123",
		"schoolName": "Oakwood Elementary",
		"schoolURN": "35672",
		"yearGroup": "2012",
		"classId": "C5",
		"className": "5B"
	},
	"timestamp": "2025-10-10T14:30:00Z"
}

```

**Multiple Events:**

May be sent as an array of events.

  

## Event Reference

  

### Standard xAPI Verbs

  
See https://registry.tincanapi.com/#home/verbs for reference

Use these standard verbs for consistent data:

  

| Verb | When to Use | Optional Properties

| `answered` | User answered a question | `questionId` |

| `launched` | User started an activity | `activityId` |

| `completed` | User completed an activity | `activityId` |

| `experienced` | User viewed content | `contentId` |

| `loggedin` | User logged in | 

| `loggedout` | User logged out | 


  

  

### User Traits

  

Traits describe who the user is (sent separately from properties, optional):

  

| Trait | Type | Description | Example |

| `role` | string | User role | `"student"` or `"teacher"` |

| `schoolName` | string | School name | `"Oakwood Elementary"` |

| `schoolURN` | string | School URN (England & Wales) | `"15463"` |

| `schoolId` | string | Your Unique school identifier | `"123"` |

| `yearGroup` | string | Year group | `"KS4"` |

| `className` | string | Class name | `"5B"` |

| `classId` | string | Class name | `"5B"` |


---

  

## API Reference  

### Response Format

  

**Success (200):**

```json

{
	"success": true,
	"statements_created": 1,
	"request_id": "uuid"
}

```

  

**Error (400/401/403/500):**

```json

{
	"error": "Error message",
	"request_id": "uuid",
	"validation_errors": [
		{
			"index": 0,
			"field": "userId",
			"message": "userId is required"
		}
	]
}

```

  


  

