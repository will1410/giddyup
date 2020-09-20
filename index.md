# [Giddyup Do-Nut](https://www.youtube.com/watch?v=nHjS06K-dew){:target="\_blank"}
(AKA "This is SPECTACULAR sql nerdery right here")

<br /><br />

The library system I work for has 51 libraries spread out over 6797 square miles.  Many of these 51 libraries are very small (most serving communities of less than 5000 people, several serving communities of less than 750 people) and only have part time library staff.  The state of Kansas supports a state wide courier system that serves all of these libraries as well as 300 other libraries throughout the state.  One of my jobs is to answer questions along the lines of "I just checked in a book and something weird is happening," or "A patron wants to pay a fine on this book, but I can't find the barcode number in the system," or "I got a book from the courier today and I can't figure out why it was shipped to me," or "Bob returned a book today but Koha says it was checked out to Jim and now it's on hold for Bob.  What happened?"

I've got lots of reports that can answer a lot of these kinds of questions, but one of the problems with running reports is keeping track of where they all are.  I have reports for request history, transfer history, circulation history, modification history, etc. etc. etc.

I also frequently have to send e-mails back and forth from library to library asking questions about items and I always try to include the following information in any e-mail I send concerning an item:

- Item homebranch:
- Current branch:
- Shelving location:
- Item type:
- Collection code:
- Call#:
- Author:
- Title:
- Item barcode:

and I like to put that information in that order because we classify all of our items in that order - home, location, type, code, call, author, title, bc.  If I'm asking a staff member at one of our libraries to look for something on the shelf, everything they need to do a shelf-check is there in that list.  Trying to copy and paste that information into an e-mail, though, is hard to do from an item's details page.  

To further complicate answering these types of questions, some of the items people question me about are items that have been deleted.  We have cron jobs running on our system that mark any item that has been overdue as "Lost" after it is 45 days overdue, another cron that deletes any "Lost" items after they have been "Lost" for more than 13 months, and another cron that purges data from the deleteditems table 13 months after the item has been deleted.  It is not unusual for staff to find items in their bookdrops that are 16 months overdue.  When they check one of those items in, I sometimes get the "When I check this in, Koha gives me a 'barcode not found' message.  What's up with that?" and when I say "It was probably deleted automatically" they often ask "Who deleted it?  When was it deleted?  Why was it deleted?  Was it lost when it was deleted?  Tell me more about this item."

So, like any lazy person, I spent hours and hours trying to come up with a way to make my life easier.  This report is the end result of those hours and hours and hours of work.  I wanted a report that merely asked for a barcode number and would then give me the classification information for that item, tell me whether the item was still in the catalog or had been deleted in the previous 13 months, would tell me whether or not the bibliographic record was still in the catalog or had been deleted within the previous 13 months, would show me any notes on the item record, would show me it's total circ count, date added, date last borrowed, date last seen, and date last modified, plus show me whether or not the item was currently checked out (and, if so, when it was due), as well as any possible status that was set for the item - lost, damaged, withdrawn, and not for loan (and I wanted the descriptions, not just the numeric codes for lost, not for loan, etc.).  If the item was currently in transit, I wanted to know that to along with where it was being shipped from and where it was being shipped to and when it was shipped.  Then I wanted links to the most common repots I'm asked about when people ask me questions about items -- the current borrower (if it's checked out), the bibliographic record, the item record, and reports for the item's circulation, in transit and request history.  And, finally, I wanted something that could easily be copied and pasted into an e-mail.

Yeah.  I wanted a lot.

This is actually the current working version of the report I wrote:

~~~sql
SELECT
  Concat_Ws('<br />',
    '<h3 style="color: white; background-color: #829356; text-align: center;">This item is currently in the catalog</h3>',
    Concat('Item homebranch: ', items.homebranch),
    Concat('Current branch: ', items.holdingbranch),
    Concat('Shelving location: ', items.location),
    Concat('Item type: ', items.itype),
    Concat('Collection code: ', ccodes.lib),
    Concat('Call#: ', items.itemcallnumber),
    Concat('Author: ', biblio.author),
    Concat('Title: ',
      Concat_Ws(' ',
        '<span style="text-transform: uppercase">',
        biblio.title,
        ExtractValue(biblio_metadata.metadata, '//datafield[@tag="245"]/subfield[@code="b"]'),
        ExtractValue(biblio_metadata.metadata, '//datafield[@tag="245"]/subfield[@code="p"]'),
        ExtractValue(biblio_metadata.metadata, '//datafield[@tag="245"]/subfield[@code="n"]'),
        '</span>')
      ),
    Concat('Item barcode: ', Upper(items.barcode)),
    Concat('<br />Public notes: ', items.itemnotes),
    Concat('Non-public notes: ', items.itemnotes_nonpublic),
    Concat('<br />Total circulation: ', (Sum((Coalesce(items.issues, 0)) + (Coalesce(items.renewals, 0))))),
    Concat('(', items.issues, ' checkouts + ', items.renewals, ' renewals)'),
    Concat('<br />Date added: ', items.dateaccessioned),
    Concat('Last borrowed: ', items.datelastborrowed),
    Concat('Last seen: ', items.datelastseen),
    Concat('Item record last modified: ', items.timestamp),
    Concat('<br />Due date: ', If(issuesi.date_due IS NULL, "-", Date_Format(issuesi.date_due, "%Y.%m.%d"))),
    Concat("Not for loan status: ", If(items.notforloan = 0, "-", If(items.notforloan IS NULL, "-", nfl.lib))),
    Concat("Damaged status: ", If(items.damaged = 0, "-", If(items.damaged IS NULL, "-", damagedi.lib))),
    Concat("Lost status: ", If(items.itemlost = 0, "-", If(items.itemlost IS NULL, "-", Concat(losti.lib, " on ", items.itemlost_on)))),
    Concat("Withdrawn status: ", If(items.withdrawn = 0, "-", If(items.withdrawn IS NULL, "- ", Concat(withdrawni.lib, " on ", items.withdrawn_on)))),
    Concat("<br /> In transit from ", If(transfersi.frombranch IS NULL, "-", Concat(transfersi.frombranch, " to ", transfersi.tobranch, " since ", transfersi.datesent))),
    Concat("<br />Link to borrower: ", If(issuesi.date_due IS NULL, "-", Concat("<a href='/cgi-bin/koha/circ/circulation.pl?borrowernumber=", issuesi.borrowernumber, "' target='_blank'>go to the borrower's account</a>"))),
    Concat("Link to title: ", Concat("<a href='/cgi-bin/koha/catalogue/detail.pl?biblionumber=", biblio.biblionumber, "' target='_blank'>go to the bibliographic record</a>")),
    Concat("Link to item: ", Concat("<a href='/cgi-bin/koha/catalogue/moredetail.pl?itemnumber=", items.itemnumber, "&biblionumber=", biblio.biblionumber, "' target='_blank'>go to the item record</a>")),
    Concat("Item circ history: ", Concat("<a href='/cgi-bin/koha/reports/guided_reports.pl?reports=2785&phase=Run+this+report&param_name=Enter+item+barcode+number&sql_params=", items.barcode, "' target='_blank'>see item circ history</a>")),
    Concat("Item action log history: ", Concat("<a href='/cgi-bin/koha/reports/guided_reports.pl?reports=3342&phase=Run+this+report&param_name=Enter+item+number&sql_params=", items.itemnumber, "' target='_blank'>see action log history</a>")),     
    Concat("Item in transit history: ", Concat("<a href='/cgi-bin/koha/reports/guided_reports.pl?reports=2784&phase=Run+this+report&sql_params=", items.barcode, "' target='_blank'>see item transit history</a>")),
    Concat("Request history on this title: ", Concat("<a href='/cgi-bin/koha/reports/guided_reports.pl?reports=3039&phase=Run+this+report&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=", biblio.biblionumber, "&sql_params=%25' target='_blank'>see title's request history</a>")),
    Concat("Request history on this item: ", Concat("<a href='/cgi-bin/koha/reports/guided_reports.pl?reports=3039&phase=Run+this+report&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=%25&sql_params=", items.barcode, "' target='_blank'>see item's request history</a>")),
    '<br /><h3 style="color: white; background-color: #829356; text-align: center;">This item is currently in the catalog<br />it has not been deleted</h3>'
  ) AS INFO
FROM
  items
  JOIN biblio
    ON items.biblionumber = biblio.biblionumber
  JOIN biblio_metadata
    ON biblio_metadata.biblionumber = biblio.biblionumber AND
      items.biblionumber = biblio_metadata.biblionumber
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'CCODE'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) ccodes
    ON items.ccode = ccodes.authorised_value
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'NOT_LOAN'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) nfl
    ON items.notforloan = nfl.authorised_value
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'DAMAGED'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) damagedi
    ON items.damaged = damagedi.authorised_value
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'LOST'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) losti
    ON items.itemlost = losti.authorised_value
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'WITHDRAWN'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) withdrawni
    ON items.withdrawn = withdrawni.authorised_value
  LEFT JOIN (
    SELECT
      branchtransfers.itemnumber,
      branchtransfers.frombranch,
      branchtransfers.datesent,
      branchtransfers.tobranch,
      branchtransfers.datearrived
    FROM
      branchtransfers
    WHERE
      branchtransfers.datearrived IS NULL
    GROUP BY
      branchtransfers.itemnumber,
      branchtransfers.frombranch,
      branchtransfers.datesent,
      branchtransfers.tobranch,
      branchtransfers.datearrived
  ) transfersi
    ON items.itemnumber = transfersi.itemnumber
  LEFT JOIN (
    SELECT
      issues.itemnumber,
      issues.date_due,
      issues.borrowernumber
    FROM
      issues
  ) issuesi
    ON items.itemnumber = issuesi.itemnumber
WHERE
  items.barcode LIKE Concat("%", @brcd := <<Enter item barcode number>> COLLATE utf8mb4_unicode_ci, "%")
GROUP BY
  items.itemnumber
UNION
SELECT
  Concat_Ws('<br />',
    '<h2 style="color: white; background-color: #AD2A1A; text-align: center;">This item has been deleted</h2>',
    Concat('At the time of its deletion on:  <ins><strong>', deleteditems.timestamp, "<br /></strong></ins> this item's information was as follows:<br />"),
    Concat('Item homebranch: ', deleteditems.homebranch),
    Concat('Current branch: ', deleteditems.holdingbranch),
    Concat('Shelving location: ', deleteditems.location),
    Concat('Item type: ', deleteditems.itype),
    Concat('Collection code: ', ccodes.lib),
    Concat('Call#: ', deleteditems.itemcallnumber),
    Concat('Author: ', Coalesce(biblio.author, deletedbiblio.author)),
    Concat('Title: ', Coalesce(biblio.title, deletedbiblio.title)),
    Concat('Item barcode: ', deleteditems.barcode),
    Concat('Replacement price: ', deleteditems.replacementprice),
    Concat('Item id number: ', deleteditems.itemnumber),
    Concat("<br />Damaged status: ", If(deleteditems.damaged = 0, "-", If(deleteditems.damaged IS NULL, "-", damagedi.lib))),
    Concat("Lost status: ", If(deleteditems.itemlost = 0, "-", If(deleteditems.itemlost IS NULL, "-", Concat(losti.lib, " on ", deleteditems.itemlost_on)))),
    Concat("Withdrawn status: ", If(deleteditems.withdrawn = 0, "-", If(deleteditems.withdrawn IS NULL, "- ", Concat(deletedwithdrawni.lib, " on ", deleteditems.withdrawn_on)))),
    If(biblio.biblionumber IS NULL, "<br />-- Bibliographic record has been deleted --", Concat("<br /><a href='/cgi-bin/koha/catalogue/detail.pl?biblionumber=", biblio.biblionumber, "' target='_blank'>go to the bibliographic record</a>")),
    Concat("<br /><a href='/cgi-bin/koha/reports/guided_reports.pl?phase=Run+this+report&reports=3009&sql_params=", Replace(Replace(Replace(Replace(Replace(Replace(Replace(deleteditems.barcode, Char(43), "%2B"), Char(47), "%2F"), Char(32), "%20"), Char(45), "%2D"), Char(36), "%24"), Char(37), "%25"), Char(46), "%2E"), "&limit=50' target='_blank'>See any fine and fees history for this item barcode number</a>"),
    '<br /><h2 style="color: white; background-color: #AD2A1A; text-align: center;">This item was deleted from the catalog<br />within the past 13 months</h2>'
  ) AS INFO
FROM
  deleteditems
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'CCODE'
    GROUP BY
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
  ) ccodes
    ON deleteditems.ccode = ccodes.authorised_value
  LEFT JOIN biblio
    ON deleteditems.biblionumber = biblio.biblionumber
  LEFT JOIN deletedbiblio
    ON deleteditems.biblionumber = deletedbiblio.biblionumber
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'DAMAGED'
  ) damagedi
    ON damagedi.authorised_value = deleteditems.damaged
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'LOST'
  ) losti
    ON losti.authorised_value = deleteditems.itemlost
  LEFT JOIN (
    SELECT
      authorised_values.category,
      authorised_values.authorised_value,
      authorised_values.lib
    FROM
      authorised_values
    WHERE
      authorised_values.category = 'WITHDRAWN'
  ) deletedwithdrawni
    ON deletedwithdrawni.authorised_value = deleteditems.withdrawn
WHERE
  deleteditems.barcode LIKE Concat("%", @brcd, "%")
GROUP BY
  deleteditems.itemnumber
~~~
