# 19. Admin Console MVP

Admin cần ba màn hình chính.

## Review Queue

- Submission ID.
- Product.
- Publisher.
- Version.
- Submitted time.
- Automated check result.
- Status.

## Submission Detail

- Listing metadata.
- Read-only workflow graph.
- JavaScript source.
- Capability.
- Target packages.
- Validation result.
- Candidate artifact hash.
- Test information.
- Review note.
- Approve, Request Changes, Reject.

## Access Control

- Search user.
- Search entitlement.
- Disable device.
- Revoke entitlement.
- Ban user.
- Suspend product.
- Xem lý do và thời điểm thao tác.

Mọi admin action cần lưu:

```text
actorId
action
targetType
targetId
reason
createdAt
```

# 20. Publisher Console MVP

Publisher chỉ cần:

- Danh sách product.
- Submission history.
- Submission status.
- Review note.
- Published version.
- Product sales cơ bản.
- Trạng thái product.

Việc sửa graph vẫn diễn ra trong Studio, không nằm trên web.
