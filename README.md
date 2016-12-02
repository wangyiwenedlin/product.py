# product.py
from extension import mysql
from flask import *
from flask.ext.login import login_required, current_user
import os
import shutil

product = Blueprint('product', __name__, template_folder='views')

@product.route('/product/delete')
@login_required
def product_delete_route():
    next = request.args.get("next")
    productid = request.args.get("id")
    version = request.args.get("version")
    db = request.args.get("db")
    cur = mysql.connection.cursor()
    cur.execute('DELETE FROM %s WHERE productid="%s" AND version="%s";'%(db,productid,version))
    mysql.connection.commit()
    return redirect("/product?id=%s&next=%s#%s"%(productid,next,db))

@product.route('/product/release')
@login_required
def product_release_route():
    next = request.args.get("next")
    productid = request.args.get("id")
    version = request.args.get("version")
    db = request.args.get("db")
    cur = mysql.connection.cursor()
    cur.execute('SELECT version FROM %s WHERE productid = "%s" AND current = 1;'%(db,productid))
    current = cur.fetchone()
    cur.execute('UPDATE %s SET released=1 WHERE productid = "%s" AND version = "%s";'%(db,productid,version))
    cur.execute('UPDATE %s SET current=1 WHERE productid = "%s" AND version = "%s";'%(db,productid,version))
    if current is not None:
        cur.execute('UPDATE %s SET current=0 WHERE productid = "%s" AND version = "%s";'%(db,productid,current[0]))
    mysql.connection.commit()
    return redirect("/product?id=%s&next=%s#%s"%(productid,next,db))

@product.route('/product/edit', methods=['GET','POST'])
@login_required
def product_edit_route():
    next = request.args.get("next")
    productid = request.args.get("id")
    cur = mysql.connection.cursor()
    cur.execute('SELECT * FROM Product WHERE productid="%s";'%productid)
    product = cur.fetchone()
    specFiles=os.listdir('static/uploads/%s/spec'%productid)
    vgbmkbFiles=os.listdir('static/uploads/%s/vgbmkb'%productid)
    programFiles=os.listdir('static/uploads/%s/program'%productid)
    wiFiles=os.listdir('static/uploads/%s/wi'%productid)
    refdocFiles=os.listdir('static/uploads/%s/refdoc'%productid)
    for file1 in specFiles:
        if file1.startswith('.'):
            specFiles.remove(file1)
    for file1 in vgbmkbFiles:
        if file1.startswith('.'):
            vgbmkbFiles.remove(file1)
    for file1 in programFiles:
        if file1.startswith('.'):
            programFiles.remove(file1)
    for file1 in wiFiles:
        if file1.startswith('.'):
            wiFiles.remove(file1)
    for file1 in refdocFiles:
        if file1.startswith('.'):
            refdocFiles.remove(file1)
    if not product:
        return render_template('404.html'), 404
    if current_user.id!=product[3]:
        flash("You do not have the permission.", "error")
        return redirect('/?search={{next}}')
    if request.method=="POST":
        submit=request.form.get('submit')
        if submit=="Update":
            name=request.form.get("productname")
            cur.execute('UPDATE Product SET name="%s" WHERE productid="%s";'%(name, productid))
            mysql.connection.commit()
        elif submit=="Upload Specification & TA":
            uploaded_files = request.files.getlist("file[]")
            for file in uploaded_files:
                file.save(os.path.join('static/uploads/%s/spec'%productid, file.filename))
                specFiles.append(file.filename)
            flash("Successfully uploaded %s(and other files)!"%uploaded_files[0].filename, "success")
        elif submit=="Upload VGB/MKB":
            uploaded_files = request.files.getlist("file[]")
            for file in uploaded_files:
                file.save(os.path.join('static/uploads/%s/vgbmkb'%productid, file.filename))
                vgbmkbFiles.append(file.filename)
            flash("Successfully uploaded %s(and other files)!"%uploaded_files[0].filename, "success")
        elif submit=="Upload Program":
            uploaded_files = request.files.getlist("file[]")
            for file in uploaded_files:
                file.save(os.path.join('static/uploads/%s/program'%productid, file.filename))
                programFiles.append(file.filename)
            flash("Successfully uploaded %s(and other files)!"%uploaded_files[0].filename, "success")
        elif submit=="Upload Work Instruction":
            uploaded_files = request.files.getlist("file[]")
            for file in uploaded_files:
                file.save(os.path.join('static/uploads/%s/wi'%productid, file.filename))
                wiFiles.append(file.filename)
            flash("Successfully uploaded %s(and other files)!"%uploaded_files[0].filename, "success")
        elif submit=="Upload Drawing":
            uploaded_files = request.files.getlist("file[]")
            print len(uploaded_files)
            for file in uploaded_files:
                print file.filename
                file.save(os.path.join('static/uploads/%s/refdoc'%productid, file.filename))
                refdocFiles.append(file.filename)
            flash("Successfully uploaded %s(and other files)!"%uploaded_files[0].filename, "success")
        elif submit=="delete":
            file_delete = request.form.get('file')
            cat = file_delete.split('/')[0]
            deletefilename = file_delete.split('/')[1]
            os.remove(os.path.join('static/uploads/%s'%productid, file_delete))
            if cat=="spec":
                specFiles.remove(deletefilename)
            elif cat=="vgbmkb":
                vgbmkbFiles.remove(deletefilename)
            elif cat=="program":
                programFiles.remove(deletefilename)
            elif cat=="wi":
                wiFiles.remove(deletefilename)
            elif cat=="refdoc":
                refdocFiles.remove(deletefilename)
        elif submit=="Delete Product":
            shutil.rmtree('static/uploads/%s'%productid)
            cur.execute('DELETE FROM Product WHERE productid="%s";'%productid)
            mysql.connection.commit()
            return redirect("/?search=%s"%next)
    cur.execute('SELECT * FROM Product WHERE productid="%s";'%productid)
    product = cur.fetchone()
    return render_template("product.html", product=product, next=next, edit=True, specFiles=specFiles, vgbmkbFiles=vgbmkbFiles, programFiles=programFiles, wiFiles=wiFiles, refdocFiles=refdocFiles)

@product.route('/_updateFilename')
@login_required
def update_route():
    productid = request.args.get("id")
    cur = mysql.connection.cursor()
    
    cur.execute('SELECT filename,version FROM MQCP WHERE productid="%s" AND released=0;'%productid)
    mqcp = cur.fetchone()
    if mqcp is not None:
        mqcp = [mqcp[0], unicode(mqcp[1])]
        
    cur.execute('SELECT filename,version FROM ProcessCard WHERE productid="%s" AND released=0;'%productid)
    processcard = cur.fetchone()
    if processcard is not None:
        processcard = [processcard[0], unicode(processcard[1])]
        
    cur.execute('SELECT filename,version FROM RecordForm WHERE productid="%s" AND released=0;'%productid)
    recordform = cur.fetchone()
    if recordform is not None:
        recordform = [recordform[0], unicode(recordform[1])]
        
    cur.execute('SELECT filename,version FROM WorkInstruction WHERE productid="%s" AND released=0;'%productid)
    workinstruction = cur.fetchone()
    if workinstruction is not None:
        workinstruction = [workinstruction[0], unicode(workinstruction[1])]
    
    cur.execute('SELECT filename,version FROM WeldingRecord WHERE productid="%s" AND released=0;'%productid)    
    weldingrecord = cur.fetchone()
    if weldingrecord is not None:
        weldingrecord = [weldingrecord[0], unicode(weldingrecord[1])]

    cur.execute('SELECT filename,version FROM DimensionalRecord WHERE productid="%s" AND released=0;'%productid)    
    dimensionalrecord = cur.fetchone()
    if dimensionalrecord is not None:
        dimensionalrecord = [dimensionalrecord[0], unicode(dimensionalrecord[1])]
    return jsonify( INFO = {"mqcp":mqcp, "processcard":processcard, "recordform":recordform, "workinstruction":workinstruction, "weldingrecord":weldingrecord, "dimensionalrecord":dimensionalrecord})
@product.route('/product')
@login_required
def product_route():
    next = request.args.get("next")
    productid = request.args.get("id")
    cur = mysql.connection.cursor()
    cur.execute('SELECT * FROM Product WHERE productid="%s";'%productid)
    product = cur.fetchone()
    if not product:
        return render_template('404.html'), 404
    if current_user.id!=product[3]:
        flash("You do not have the permission.", "error")
        return redirect('/?search={{next}}')
    
    cur.execute('SELECT * FROM MQCP WHERE productid="%s" AND released=0;'%productid)
    mqcp = cur.fetchone()
    
    cur.execute('SELECT * FROM ProcessCard WHERE productid="%s" AND released=0;'%productid)
    processcard = cur.fetchone()
    
    cur.execute('SELECT * FROM RecordForm WHERE productid="%s" AND released=0;'%productid)
    recordform = cur.fetchone()
    
    cur.execute('SELECT * FROM WorkInstruction WHERE productid="%s" AND released=0;'%productid)
    workinstruction = cur.fetchone()
    
    cur.execute('SELECT * FROM WeldingRecord WHERE productid="%s" AND released=0;' % productid)
    weldingrecord = cur.fetchone()

    cur.execute('SELECT * FROM DimensionalRecord WHERE productid="%s" AND released=0;' % productid)
    dimensionalrecord = cur.fetchone()
    
    cur.execute('SELECT * FROM MQCP WHERE productid="%s" AND released=1 ORDER BY version DESC;'%productid)
    mqcps = cur.fetchall()
    
    cur.execute('SELECT * FROM ProcessCard WHERE productid="%s" AND released=1 ORDER BY version DESC;'%productid)
    processcards = cur.fetchall()
    
    cur.execute('SELECT * FROM RecordForm WHERE productid="%s" AND released=1 ORDER BY version DESC;'%productid)
    recordforms = cur.fetchall()
    
    cur.execute('SELECT * FROM WorkInstruction WHERE productid="%s" AND released=1 ORDER BY version DESC;'%productid)
    workinstructions = cur.fetchall()
    
    cur.execute('SELECT * FROM WeldingRecord WHERE productid="%s" AND released=1 ORDER BY version DESC;' % productid)
    weldingrecords = cur.fetchall()

    cur.execute('SELECT * FROM DimensionalRecord WHERE productid="%s" AND released=1 ORDER BY version DESC;' % productid)
    dimensionalrecords = cur.fetchall()
    
    return render_template("product.html", product=product, next=next, edit=False, mqcp=mqcp, processcard=processcard, recordform=recordform, workinstruction=workinstruction, weldingrecord=weldingrecord, dimensionalrecord=dimensionalrecord, mqcps=mqcps, processcards=processcards, recordforms=recordforms, workinstructions=workinstructions,weldingrecords=weldingrecords, dimensionalrecords=dimensionalrecords)

