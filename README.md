/*
 * CAMPUS LOST-AND-FOUND MANAGEMENT SYSTEM - PROTOTYPE 2.3
 * Uses arrays, strings, functions, pointers, loops and conditions.
 * No structures or file handling. Data is temporary until exit.
 */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>
#include <limits.h>

#define MAX_RECORDS 100
#define NAME_LEN 50
#define CATEGORY_LEN 30
#define LOCATION_LEN 60
#define DESCRIPTION_LEN 120
#define DATE_LEN 15
#define CONTACT_LEN 20
#define NOTE_LEN 120

/* Parallel arrays: matching indexes belong to the same report. */
int itemId[MAX_RECORDS], reportType[MAX_RECORDS], itemStatus[MAX_RECORDS];
char itemName[MAX_RECORDS][NAME_LEN];
char category[MAX_RECORDS][CATEGORY_LEN];
char location[MAX_RECORDS][LOCATION_LEN];
char description[MAX_RECORDS][DESCRIPTION_LEN];
char reportDate[MAX_RECORDS][DATE_LEN];
char contact[MAX_RECORDS][CONTACT_LEN];
char statusNote[MAX_RECORDS][NOTE_LEN];
int recordCount = 0, nextId = 1001;

void readLine(const char *prompt, char output[], int size)
{
    int ch;
    size_t len;
    printf("%s", prompt);
    if (fgets(output, size, stdin) == NULL) { output[0] = '\0'; return; }
    len = strlen(output);
    if (len > 0 && output[len - 1] == '\n') output[len - 1] = '\0';
    else while ((ch = getchar()) != '\n' && ch != EOF) { }
}

int readIntRange(const char *prompt, int min, int max)
{
    char line[50], *end;
    long value;
    while (1) {
        readLine(prompt, line, sizeof(line));
        if (line[0] == '\0') { printf("Input cannot be empty.\n"); continue; }
        value = strtol(line, &end, 10);
        while (isspace((unsigned char)*end)) end++;
        if (end != line && *end == '\0' && value >= min && value <= max)
            return (int)value;
        printf("Enter a whole number from %d to %d.\n", min, max);
    }
}

void readRequiredText(const char *prompt, char output[], int size)
{
    do {
        readLine(prompt, output, size);
        if (output[0] == '\0') printf("This field cannot be empty. Try again.\n");
    } while (output[0] == '\0');
}

int containsIgnoreCase(const char text[], const char query[])
{
    int i, j;
    if (query[0] == '\0') return 1;
    for (i = 0; text[i] != '\0'; i++) {
        for (j = 0; query[j] != '\0'; j++) {
            if (text[i + j] == '\0' ||
                tolower((unsigned char)text[i + j]) != tolower((unsigned char)query[j])) break;
        }
        if (query[j] == '\0') return 1;
    }
    return 0;
}

int equalIgnoreCase(const char a[], const char b[])
{
    int i = 0;
    while (a[i] && b[i]) {
        if (tolower((unsigned char)a[i]) != tolower((unsigned char)b[i])) return 0;
        i++;
    }
    return a[i] == '\0' && b[i] == '\0';
}

/* Every word in the query must appear somewhere in the combined record text.
   This allows a search such as "pink box" to match name="Box" and
   description="Pink colour". */
int allQueryWordsFound(const char searchableText[], const char query[])
{
    char queryCopy[LOCATION_LEN];
    char *word;
    size_t length = strlen(query);

    if (length >= sizeof(queryCopy)) return 0;
    strcpy(queryCopy, query);

    word = strtok(queryCopy, " \t");
    if (word == NULL) return 0;

    while (word != NULL) {
        if (!containsIgnoreCase(searchableText, word)) return 0;
        word = strtok(NULL, " \t");
    }
    return 1;
}

const char *typeName(int type) { return type == 1 ? "LOST" : "FOUND"; }
const char *statusName(int status)
{
    if (status == 1) return "ACTIVE";
    if (status == 2) return "RETURNED";
    return "CLOSED";
}

void displayRecord(int i)
{
    printf("\n---------------------------------------------\n");
    printf("ID: %d | Type: %s | Status: %s\n", itemId[i], typeName(reportType[i]), statusName(itemStatus[i]));
    printf("Item: %s\nCategory: %s\nLocation: %s\nDate: %s\n", itemName[i], category[i], location[i], reportDate[i]);
    printf("Description: %s\nContact: %s\n", description[i], contact[i]);
    printf("Latest status comment: %s\n", statusNote[i][0] ? statusNote[i] : "No comment");
    printf("---------------------------------------------\n");
}

int findIndexById(int id)
{
    int i;
    for (i = 0; i < recordCount; i++) if (itemId[i] == id) return i;
    return -1;
}

int possibleDuplicate(int type, const char name[], const char place[])
{
    int i;
    for (i = 0; i < recordCount; i++)
        if (reportType[i] == type && itemStatus[i] == 1 &&
            equalIgnoreCase(itemName[i], name) && equalIgnoreCase(location[i], place)) return i;
    return -1;
}

void checkNewRecordMatches(int newIndex);

void addRecord(int type)
{
    int i, duplicate;
    char answer[10];
    if (recordCount >= MAX_RECORDS) { printf("Maximum record limit reached.\n"); return; }
    printf("\n--- %s ITEM REGISTRATION ---\n", type == 1 ? "LOST" : "FOUND");
    readRequiredText("Item name: ", itemName[recordCount], NAME_LEN);
    readRequiredText("Category: ", category[recordCount], CATEGORY_LEN);
    readRequiredText("Location: ", location[recordCount], LOCATION_LEN);
    readRequiredText("Date (DD-MM-YYYY): ", reportDate[recordCount], DATE_LEN);
    readRequiredText("Description: ", description[recordCount], DESCRIPTION_LEN);
    readRequiredText("Contact number: ", contact[recordCount], CONTACT_LEN);
    duplicate = possibleDuplicate(type, itemName[recordCount], location[recordCount]);
    if (duplicate != -1) {
        printf("Similar active report exists with ID %d.\n", itemId[duplicate]);
        readLine("Add as a separate report anyway? (Y/N): ", answer, sizeof(answer));
        if (answer[0] != 'Y' && answer[0] != 'y') { printf("Registration cancelled.\n"); return; }
    }
    i = recordCount;
    itemId[i] = nextId++;
    reportType[i] = type;
    itemStatus[i] = 1;
    strcpy(statusNote[i], "New report registered");
    recordCount++;
    printf("Report registered. Your Item ID is %d.\n", itemId[i]);
    checkNewRecordMatches(i);
}

void searchRecords(void)
{
    char query[LOCATION_LEN];
    int choice, i, found = 0;
    if (recordCount == 0) { printf("No records available.\n"); return; }
    printf("\nSEARCH BY\n1. All item details (name, category, location, description)\n");
    printf("2. Item name\n3. Category\n4. Location\n5. Item ID\n6. Status\n");
    choice = readIntRange("Choice: ", 1, 6);
    if (choice == 5) {
        int id = readIntRange("Item ID: ", 1, INT_MAX), index = findIndexById(id);
        if (index < 0) printf("No record found.\n"); else displayRecord(index);
        return;
    }
    if (choice == 6) {
        int status = readIntRange("Status (1 Active, 2 Returned, 3 Closed): ", 1, 3);
        for (i = 0; i < recordCount; i++) if (itemStatus[i] == status) { displayRecord(i); found++; }
    } else {
        readRequiredText("Search text: ", query, sizeof(query));
        for (i = 0; i < recordCount; i++) {
            int match = 0;
            if (choice == 1) {
                char searchableText[NAME_LEN + CATEGORY_LEN + LOCATION_LEN + DESCRIPTION_LEN + 4];
                snprintf(searchableText, sizeof(searchableText), "%s %s %s %s",
                         itemName[i], category[i], location[i], description[i]);
                match = allQueryWordsFound(searchableText, query);
            } else if (choice == 2) {
                match = containsIgnoreCase(itemName[i], query);
            } else if (choice == 3) {
                match = containsIgnoreCase(category[i], query);
            } else {
                match = containsIgnoreCase(location[i], query);
            }
            if (match) { displayRecord(i); found++; }
        }
    }
    printf("\nMatching records: %d\n", found);
}

void viewAndFilterRecords(void)
{
    int choice, i, found = 0;
    if (recordCount == 0) { printf("No records available.\n"); return; }
    printf("\n1. All records\n2. Lost only\n3. Found only\n4. Active only\n5. Returned only\n6. Closed only\n");
    choice = readIntRange("Choice: ", 1, 6);
    for (i = 0; i < recordCount; i++) {
        int include = choice == 1 ||
            (choice == 2 && reportType[i] == 1) ||
            (choice == 3 && reportType[i] == 2) ||
            (choice == 4 && itemStatus[i] == 1) ||
            (choice == 5 && itemStatus[i] == 2) ||
            (choice == 6 && itemStatus[i] == 3);
        if (include) { displayRecord(i); found++; }
    }
    printf("Records displayed: %d\n", found);
}

int matchScore(int lost, int found)
{
    int score = 0;

    /* Name contributes up to 40 points. */
    if (equalIgnoreCase(itemName[lost], itemName[found])) score += 40;
    else if (containsIgnoreCase(itemName[lost], itemName[found]) ||
             containsIgnoreCase(itemName[found], itemName[lost])) score += 28;

    /* Category contributes up to 20 points. */
    if (equalIgnoreCase(category[lost], category[found])) score += 20;
    else if (containsIgnoreCase(category[lost], category[found]) ||
             containsIgnoreCase(category[found], category[lost])) score += 10;

    /* Location contributes up to 15 points. */
    if (equalIgnoreCase(location[lost], location[found])) score += 15;
    else if (containsIgnoreCase(location[lost], location[found]) ||
             containsIgnoreCase(location[found], location[lost])) score += 8;

    /* Description contributes up to 25 points. */
    if (equalIgnoreCase(description[lost], description[found])) score += 25;
    else if (containsIgnoreCase(description[lost], description[found]) ||
             containsIgnoreCase(description[found], description[lost])) score += 15;

    return score;
}

void updateStatusForIndex(int index)
{
    int status;
    char note[NOTE_LEN];
    displayRecord(index);
    printf("1. Active\n2. Returned\n3. Closed\n");
    status = readIntRange("New status: ", 1, 3);
    readRequiredText("Add a comment (e.g. awaiting owner, item found, returned to owner): ", note, sizeof(note));
    itemStatus[index] = status;
    strcpy(statusNote[index], note);
    printf("Report %d updated: %s. Comment saved.\n", itemId[index], statusName(status));
}

void quickUpdateMatchedPair(int lostIndex, int foundIndex)
{
    int choice;
    char confirm[10], note[NOTE_LEN];

    printf("\n--- QUICK MATCH STATUS UPDATE ---\n");
    printf("1. Mark BOTH reports as RETURNED (only after owner verification)\n");
    printf("2. Update LOST report only\n");
    printf("3. Update FOUND report only\n");
    printf("4. Skip for now\n");
    choice = readIntRange("Choose an action: ", 1, 4);

    if (choice == 1) {
        readLine("Have you verified that the item was returned to its owner? (Y/N): ", confirm, sizeof(confirm));
        if (confirm[0] != 'Y' && confirm[0] != 'y') {
            printf("No status changed. Verify the item before marking it returned.\n");
            return;
        }
        readRequiredText("Comment for both reports: ", note, sizeof(note));
        itemStatus[lostIndex] = 2;
        itemStatus[foundIndex] = 2;
        strcpy(statusNote[lostIndex], note);
        strcpy(statusNote[foundIndex], note);
        printf("Both reports marked RETURNED and comments saved.\n");
    } else if (choice == 2) {
        updateStatusForIndex(lostIndex);
    } else if (choice == 3) {
        updateStatusForIndex(foundIndex);
    } else {
        printf("No changes made. You can update the reports later from the main menu.\n");
    }
}

/* Compare a newly registered report with existing active reports of the
   opposite type and immediately alert the user about likely matches. */
void checkNewRecordMatches(int newIndex)
{
    int i, lostIndex, foundIndex, score, matches = 0;
    int matchedLost[MAX_RECORDS], matchedFound[MAX_RECORDS];

    if (itemStatus[newIndex] != 1) return;

    for (i = 0; i < recordCount; i++) {
        if (i == newIndex || itemStatus[i] != 1 ||
            reportType[i] == reportType[newIndex]) continue;

        if (reportType[newIndex] == 1) {
            lostIndex = newIndex;
            foundIndex = i;
        } else {
            lostIndex = i;
            foundIndex = newIndex;
        }

        score = matchScore(lostIndex, foundIndex);
        if (score >= 50) {
            if (matches == 0) {
                printf("\n========== POSSIBLE MATCH ALERT ==========\n");
                printf("A newly registered report may match an existing report.\n");
            }
            if (matches < MAX_RECORDS) {
                matchedLost[matches] = lostIndex;
                matchedFound[matches] = foundIndex;
            }
            matches++;
            printf("\nPossible match %d | Similarity: %d/100\n", matches, score);
            printf("--- LOST REPORT ---\n");
            displayRecord(lostIndex);
            printf("--- FOUND REPORT ---\n");
            displayRecord(foundIndex);
        }
    }

    if (matches > 0) {
        printf("Please verify the item manually before updating its status.\n");
        printf("The two reports remain separate records.\n");
        printf("==========================================\n");
        if (matches == 1) {
            quickUpdateMatchedPair(matchedLost[0], matchedFound[0]);
        } else {
            int selected;
            printf("\nMore than one possible match was found.\n");
            selected = readIntRange("Enter the match number to quick-update (0 to skip): ", 0, matches);
            if (selected > 0 && selected <= MAX_RECORDS)
                quickUpdateMatchedPair(matchedLost[selected - 1], matchedFound[selected - 1]);
        }
    } else {
        printf("No strong match was detected for this new report.\n");
    }
}

void findMatches(void)
{
    int i, j, score, count = 0;
    printf("\n--- POSSIBLE LOST-FOUND MATCHES ---\n");
    for (i = 0; i < recordCount; i++) {
        if (reportType[i] != 1 || itemStatus[i] != 1) continue;
        for (j = 0; j < recordCount; j++) {
            if (reportType[j] == 2 && itemStatus[j] == 1) {
                score = matchScore(i, j);
                if (score >= 50) {
                    printf("\nPossible match %d | Score: %d/100\n", ++count, score);
                    printf("LOST : ID %d - %s (%s)\n", itemId[i], itemName[i], location[i]);
                    printf("FOUND: ID %d - %s (%s)\n", itemId[j], itemName[j], location[j]);
                    printf("Manually verify the description and contact details.\n");
                }
            }
        }
    }
    if (count == 0) printf("No possible matches found.\n");
    printf("Similarity uses name, category, location and description.\n");
    printf("A score is only a suggestion, not proof of identity.\n");
}

void updateStatus(void)
{
    int id, index;
    if (recordCount == 0) { printf("No records to update.\n"); return; }
    id = readIntRange("Enter Item ID: ", 1, INT_MAX);
    index = findIndexById(id);
    if (index < 0) { printf("Item ID not found.\n"); return; }
    updateStatusForIndex(index);
}

void editRecord(void)
{
    int id, index, choice;
    if (recordCount == 0) { printf("No records to edit.\n"); return; }
    id = readIntRange("Enter Item ID to edit: ", 1, INT_MAX);
    index = findIndexById(id);
    if (index < 0) { printf("Item ID not found.\n"); return; }
    while (1) {
        displayRecord(index);
        printf("\nEDIT FIELD\n1. Name\n2. Category\n3. Location\n4. Date\n5. Description\n6. Contact\n7. Finish\n");
        choice = readIntRange("Choice: ", 1, 7);
        if (choice == 1) readRequiredText("New name: ", itemName[index], NAME_LEN);
        else if (choice == 2) readRequiredText("New category: ", category[index], CATEGORY_LEN);
        else if (choice == 3) readRequiredText("New location: ", location[index], LOCATION_LEN);
        else if (choice == 4) readRequiredText("New date: ", reportDate[index], DATE_LEN);
        else if (choice == 5) readRequiredText("New description: ", description[index], DESCRIPTION_LEN);
        else if (choice == 6) readRequiredText("New contact: ", contact[index], CONTACT_LEN);
        else break;
        printf("Record field updated.\n");
    }
}

void showDashboard(void)
{
    int i, lost = 0, found = 0, active = 0, returned = 0, closed = 0;
    for (i = 0; i < recordCount; i++) {
        if (reportType[i] == 1) lost++; else found++;
        if (itemStatus[i] == 1) active++;
        else if (itemStatus[i] == 2) returned++;
        else closed++;
    }
    printf("\n========== SYSTEM DASHBOARD ==========\n");
    printf("Total reports        : %d\n", recordCount);
    printf("Lost reports         : %d\n", lost);
    printf("Found reports        : %d\n", found);
    printf("Active reports       : %d\n", active);
    printf("Returned items       : %d\n", returned);
    printf("Closed reports       : %d\n", closed);
    printf("Available record slots: %d\n", MAX_RECORDS - recordCount);
    printf("======================================\n");
}

void displayMenu(void)
{
    printf("\n============================================\n");
    printf("       CAMPUS LOST-AND-FOUND SYSTEM - V2.3\n");
    printf("============================================\n");
    printf("1. Report a Lost Item\n2. Report a Found Item\n3. Search for Items\n");
    printf("4. View and Filter Records\n5. Find Possible Matches\n");
    printf("6. Update Item Status\n7. Edit Existing Record\n");
    printf("8. View Dashboard / Statistics\n9. Exit\n");
    printf("============================================\n");
}

int main(void)
{
    int choice;
    printf("\nWelcome to the Campus Lost-and-Found System!\n");
    showDashboard();
    do {
        displayMenu();
        choice = readIntRange("Enter your choice: ", 1, 9);
        switch (choice) {
            case 1: addRecord(1); break;
            case 2: addRecord(2); break;
            case 3: searchRecords(); break;
            case 4: viewAndFilterRecords(); break;
            case 5: findMatches(); break;
            case 6: updateStatus(); break;
            case 7: editRecord(); break;
            case 8: showDashboard(); break;
            case 9: printf("Thank you for using the system. Goodbye!\n"); break;
        }
    } while (choice != 9);
    return 0;
}
