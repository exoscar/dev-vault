
ALTER TABLE issue
ADD COLUMN search_vector tsvector;

Create GIN Index
CREATE INDEX idx_issue_search
ON issue USING GIN(search_vector);

UPDATE issue
SET search_vector =
    to_tsvector(
        'english',
        coalesce(title, '') || ' ' ||
        coalesce(description, '')
    );



ALTER TABLE project
ADD COLUMN search_vector tsvector;

UPDATE project
SET search_vector =
    to_tsvector(
        'english',
        coalesce(name, '') || ' ' ||
        coalesce(description, '')
    );

CREATE INDEX idx_project_search_vector
ON project
USING GIN(search_vector);